[Anterior](05-auth.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](07-ci.md)

# Exigindo Autenticação

Agora que implementamos a lógica de login, precisamos aplicar ela ao nosso serviço. Atualmente nosso serviço possui 4 conjuntos de rotas, `/auth/`, `/api/`, `/ping`,  `/~/ready`. Dessas rotas somente uma precisa de autenticação (`api`) enquanto as outras servem para fazer a autenticação (`auth`), ver a saúde do serviço (`ping`) e ver a disponibildiade de receber chamadas do serviço (`ready`). Assim, precisamos implementar "algo" que vai aplicar o sistema de login somente a rota `/api`. Esse algo será um middleware que nós vamos construir, ao contrário dos outros que já utilizamos, e lidará com a lógica de autenticação.

> **Middleware**
>
> Já utilizamos Middlewares anterioemente, mas como foram utilizações superficiais não foi preciso entender mais a fundo o que eram. Agora creio que seja um momento interessante de defini-los. Middlewares não são um conceito exclusivo de aplicações web, podendo ser utilizados tanto em sistemas operacionais como em programas do dia a dia. Os middlewares proveem um conjunto de serviços e capacidades comuns a uma aplicação que sua base não prove. Esses serviços e capacidades podem ser gerenciamento de dados, tratamento de mensagens, autenticação e logs. Assim, middlewares atual como um tecido conectivo de vários serviços da aplicação.
>
> No caso de middlewares de aplicações web, geralmente sua funcionalidade é adicionar comportamentos aos processamento de request e de response. Eles consegue se conectar a um request, ou a um response, que chegou ao servidor e alterar este request, inclusive respondendo antes do esperado, ou alterando o response. Utilizamos o middleware de `Logger` para incluir logs ao processamento do nosso request e o middleware de `DefaultHeaders` para incluir um header em nosso response.

As alterações a seguir nos exigiram modificar o Cargo.toml para conter a crate `futures` e mover a crate `actix-server` para `dependencies`:

```toml
[dependencies]
actix = "0.9.0"
#...
jsonwebtokens = "1.0.0-alpha.12"
actix-service = "1.0.5"
futures = "0.3.4"

[dev-dependencies]
bytes = "0.5.3"
```

## Estrutura de um middleware com Actix

Um middleware pode ser registrado em cada `App`, `scope` ou `Resource` do servidor e é executado em ordem oposta a seu registro. De modo geral, middlewares em Actix são um tipo, preferenciamente uma struct, que implementa as traits `Service` e `Transform`, da crate `actix_service`. Assim, cada um dos métodos da trait tem a capacidade de responder algo imediatamente ou através de uma future. O exemplo mais básico de Middleware seria um `hello world` no request (`Hello from Request`) e outro na response (`Hello from Response`), conforme o código a seguir:

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

use actix_service::{Service, Transform};
use actix_web::{dev::ServiceRequest, dev::ServiceResponse, Error};
use futures::future::{ok, Ready};
use futures::Future;

pub struct SayHi;

impl<S, B> Transform<S> for SayHi
where
    S: Service<Request = ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Request = ServiceRequest;
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = SayHiMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ok(SayHiMiddleware { service })
    }
}

pub struct SayHiMiddleware<S> {
    service: S,
}

impl<S, B> Service for SayHiMiddleware<S>
where
    S: Service<Request = ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Request = ServiceRequest;
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>>>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.service.poll_ready(cx)
    }

    fn call(&mut self, req: ServiceRequest) -> Self::Future {
        println!("Hello from Request. You requested: {}", req.path());

        let fut = self.service.call(req);

        Box::pin(async move {
            let res = fut.await?;

            println!("Hello from Response");
            Ok(res)
        })
    }
}
```

Existem 2 passos no processamento de um middleware. O primeiro é sua inicialização, na qual o `middleware factory` é chamado com o próximo serviço encadeado. Isso corresponde a função `new_transform` da trait `Transform` com o tipo `S` definido como `Service<_,_,_>` sendo o campo`service` que `SayHiMiddleware<S>` implementa. Depois disso o método `call` é chamado com o request. `poll_ready` nos indica quando a future está pronta.

O parâmetro `req: ServiceRequest` pode ser passado de forma mutável e adaptar o request, por exemplo um request do tipo `application/edn` poderia ser convertido para `application/json`/ Depois disso, temos a future contendo a response em `let fut = self.service.call(req);`, que só é concretizada dentro de uma `Box` não movível contendo um bloco `async`. a função `Pin` torna a `Box` um ponteiro fixo em memória, não movível. Depois disso, basta responde a `future` com a respostas `res`.

## Definindo o Middleware de autenticação

A primeira coisa que devemos fazer aqui é criar o módulo middleware em nosso código. Esse módulo estará contido em `src/todo_api_web/middleware/mod.rs`. Com o módulo criado podemos definir a struct de autenticação, `Authentication`, que corresponde a struct `SayHi`. É essa struct que vamos passar como argumento para o `wrap` de `App`, `App:new().wrap(crate::todo_api_web::middleware::Authentication)`. `Authentication` criará o `middleware factory` para passarmos o serviço a um middleware de fato, `AuthenticationMiddleware`,  que nos permitirá alterar o request. Este bloco de código fica assim:

```rust
use actix_service::{Service, Transform};
use actix_web::{
    dev::{ServiceRequest, ServiceResponse},
    Error, HttpResponse,
};
use futures::{
    future::{ok, Ready},
    Future,
};
use std::{
    pin::Pin,
    task::{Context, Poll},
};

use crate::{
    todo_api_web::model::http::Clients,
    todo_api::{
        core::decode_jwt,
        model::core::JwtValue,
    }
};

pub struct Authentication;

impl<S: 'static, B> Transform<S> for Authentication
where
    S: Service<Request = ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Request = ServiceRequest;
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = AuthenticationMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ok(AuthenticationMiddleware { service })
    }
}
pub struct AuthenticationMiddleware<S> {
    service: S,
}

impl<S: 'static, B> Service for AuthenticationMiddleware<S>
where
    S: Service<Request = ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Request = ServiceRequest;
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>>>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.service.poll_ready(cx)
    }

    fn call(&mut self, req: ServiceRequest) -> Self::Future {
        // ...
    }
}
```

Não há muito o que alterar aqui além do nome das structs, pois este é um modelo padrão para se gerar um middleware, porém poderiámos retornar informações ainda no nível de `new_transform` e de `poll_ready`. Já a função `call` é o centro de nossa atenção, sendo ela responsável pela manipulação de dados que queremos fazer. O primeiro caso que vamos ver é o fato de querermos que este middleware atue somente nas rotas `/api/`, assim temos duas soluções para isso. A primeira seria adicionar este middleware diretamente em `web::scope("api/")` do arquivo de rotas:

```rust
pub fn app_routes(config: &mut web::ServiceConfig) {
    config.service(
        web::scope("/")
            .service(
                web::scope("api/")
                    .wrap(crate::todo_api_web::middleware::Authentication)
                    .route("create", web::post().to(create_todo))
                    .route("index", web::get().to(show_all_todo)),
            )
            .service(
                web::scope("auth/")
                    .route("signup", web::post().to(signup_user))
                    .route("login", web::post().to(login))
                    .route("logout", web::delete().to(logout)),
            )
            .route("ping", web::get().to(pong))
            .route("~/ready", web::get().to(readiness))
            .route("", web::get().to(|| HttpResponse::NotFound())),
    );
}
```

Nnao gosto muito desta alternativa pois ela impacta a testabilidade. Portanto, prefiro a segunda alternativa que é criar uma condicional que verifica se a rota do request começa com os `scope`  que queremos. Caso a condicional for verdadeira, aplicamos nossa lógica, senão, simplesmente damos sequência ao request:

```rust
fn call(&mut self, req: ServiceRequest) -> Self::Future {
        if req.path().starts_with("/api/") {
         // ...
        } else {
            let fut = self.service.call(req);
            Box::pin(async move {
                let res = fut.await?;
                Ok(res)
            })
        }
    }
```

O `if` que definimos extrai o `path`, rota, de `ServiceRequest`, e aplica a função `starts_with` com o início da rota que queremos, `/api/`. Com isso, toda as rotas do serviço que começarem com `/api/` serão alteradas por este middleware. O próximo passo é extrairmos o header `x-auth` dos headers do request:

```rust
fn call(&mut self, req: ServiceRequest) -> Self::Future {
        if req.path().starts_with("/api/") {
            // ...
            let jwt = req.headers().get("x-auth");

            match jwt {
                None => Box::pin(async move {
                    Ok::<_,actix_http::error::Error>(req.into_response(
                        HttpResponse::BadRequest()
                        .json("{\"error\": \"x-auth is required\"}")
                        .into_body()
                    ))
                }),
                Some(token) => {
                    // ...
                }
            }
        }
        // ...
}
```

Exatraimos o header `x-auth` aplicando a função `headers` a request, que obtém todos os headers, e posteriormente escolhendo um header específico com `get`. O retorno desta função é um `Option<String>` contendo a String de Jwt. Como este campo é obrigatório, fazemos um `match` em `jwt` e no caso `None` retornamos um `BadRequest` com a informação que `x-auth` é requerido, `x-auth is required`. Depois disso precisamos de duas coisas, decodificar o token Jwt e enviar a resposta decodificada para validar ela no banco de dados, assim precisaremos da função `decode_jwt` para decodificar o `jwt` e de `Clients` armazenado em `data` para comunicar com o banco de dados: 

```rust
fn call(&mut self, req: ServiceRequest) -> Self::Future {
        if req.path().starts_with("/api/") {
            let data = req.app_data::<Clients>().expect("Failed to parse app_data");
            let jwt = req.headers().get("x-auth");

            match jwt {
                None => // ...
                Some(token) => {
                    let decoded_jwt: JwtValue = serde_json::from_value(decode_jwt(token.to_str().unwrap())).expect("Failed to parse Jwt");
                    let valid_jwt = data.postgres.send(decoded_jwt);

                    let fut = self.service.call(req);
                    Box::pin(async move {
                        match valid_jwt.await {
                            Ok(true) => {
                                let res = fut.await?;
                                Ok(res)
                            },
                            _ => {
                                Err(Error::from(()))
                            }
                        }
                    })
                }
            }
        } 
        // ...
    }
```

> Para recordar `decode_jwt`
> ```rust
> pub fn decode_jwt(jwt: &str) -> Value {
>    use jsonwebtokens::raw::{decode_json_token_slice, split_token, TokenSlices};
>
>    let TokenSlices { claims, .. } = split_token(jwt).unwrap();
>    let claims = decode_json_token_slice(claims).expect("Failed to decode token");
>    claims
>}
>```

Nossa função `call` transforma o token em uma struct `JwtValue`, que corresponde aos campos presentes no token, através da função `serde_json::from_value`. O valor de `JwtValue` é enviado para o banco de dados através de `data.postgres.send(decoded_jwt)`. Com o resultado da troca de mensagens com `DbExecutor` via `data.postgres.send` recebemos um tipo `Result<bool, MailBoxError>` e fazemos `match`. O único caso que nos interessa é o `Ok(true)`, para todos os outros lançamos uma erro. Este erro retornará `InternalServerError`, pois não podemos reutilizar o conteúdo de `req` já que foi utilizado em `let fut = self.service.call(req);` para conretizar o request. Caso o resultado do `match` seja `Ok(true)`, deixamos o request prosseguir. Agora, vamos a implementação de `JwtValue`:

```rust
#[derive(Serialize, Deserialize, Debug)]
pub struct JwtValue {
    pub id: String,
    pub email: String,
    pub expires_at: chrono::NaiveDateTime,
}

impl Message for JwtValue {
    type Result = bool;
}

impl Handler<JwtValue> for DbExecutor {
    type Result = bool;

    fn handle(&mut self, msg: JwtValue, _: &mut Self::Context) -> Self::Result {
        use crate::todo_api::db::auth::token_is_valid;

        let user = token_is_valid(&msg, &self.0.get().expect("Failed to open connection"));
        match user {
            Err(_) => false,
            Ok(user) => {
                match (user.is_active, validate_jwt_date(user.expires_at), user.id.to_string() == msg.id) {
                    (true, true, true) => true,
                    _ => false
                }
            }
        }
    }
}
```

Nosso token Jwt possui 3 campos em seu `claim` `id, email, expires_at`, assim a implementação de `JwtValue` possui estes 3 campos, definidos como `String, String, chrono::NaiveDateTime`, respectivamente. Depois definimos a trait `Message`, com o tipo `type Result = bool;`. Para `Handler`, procuramos o `User` com `token_is_valid(&msg, &self.0.get().expect("Failed to open connection"))` e depois aplicamos `match` a sua resposta. Em caso de `Err`, retornamos `false`, e em caso de `Ok`, verificamos todas as condições que queremos em outro `match` (poderia ser um `if/else`, mas creio que o match ficou mais elegante devido ao uso da tupla). Caso todos os itens da tupla, `(user.is_active, validate_jwt_date(user.expires_at), user.id.to_string() == msg.id)` sejam verdadeiros, retornamos `true`, senão `false`. Os itens da tupla são:

1. Usuário está ativo com `user.is_active`.
2. Data atual é inferior a data `expires_at` do token com `validate_jwt_date(user.expires_at)`.
3. O id do token é o mesmo do usuário cadastrado com `user.id.to_string() == msg.id`.

Quanto a nossa função `token_is_valid`:

```rust
pub fn token_is_valid(token: &JwtValue, conn: &PgConnection) -> Result<User, DbError> {
    use crate::schema::auth_user::dsl::*;

    let items = auth_user.filter(email.eq(&token.email)).load::<User>(conn);

    match items {
        Ok(users) if users.len() > 1 => Err(DbError::DatabaseConflit),
        Ok(users) if users.len() < 1 => Err(DbError::CannotFindUser),
        Ok(users) => Ok(users.first().unwrap().clone().to_owned()),
        Err(_) => Err(DbError::CannotFindUser),
    }
}
```

Note que ela é praticamente igual a função `scan_user`. Assim, podemos substituir ela por scan user, obtendo o campo `email` de `JwtValue`:

```rust
impl Handler<JwtValue> for DbExecutor {
    type Result = bool;

    fn handle(&mut self, msg: JwtValue, _: &mut Self::Context) -> Self::Result {
        use crate::todo_api::db::auth::scan_user;

        let user = scan_user(String::from(&msg.email), &self.0.get().expect("Failed to open connection"));
        match user {
            Err(_) => false,
            Ok(user) => {
                match (user.is_active, validate_jwt_date(user.expires_at), user.id.to_string() == msg.id) {
                    (true, true, true) => true,
                    _ => false
                }
            }
        }
    }
}
```

Com tudo isso pronto, basta adicionar o middleware em `App::new()`:

```rust
#[actix_rt::main]
async fn web_main() -> Result<(), std::io::Error> {
    HttpServer::new(|| {
        App::new()
        .data(Clients::new())
        .wrap(DefaultHeaders::new().header("x-request-id", Uuid::new_v4().to_string()))
        .wrap(Logger::new("IP:%a DATETIME:%t REQUEST:\"%r\" STATUS: %s DURATION:%D X-REQUEST-ID:%{x-request-id}o"))
        .wrap(crate::todo_api_web::middleware::Authentication)
        .configure(app_routes)
    })
    .workers(num_cpus::get() + 2)
    .bind("0.0.0.0:4000")
    .unwrap()
    .run()
    .await
}
```

### Testando o middleware

Nosso middleware funciona bem, e basta executar o comando `make run` para se divertir com ele, porém não temos nenhum teste que garante o comportamento do middleware. Assim, podemos criar pelo menos dois testes. O primeiro teste é não enviar um header `x-auth` para uma rota `/api/` e o segundo teste é enviar um token aleatório. Como `decode_token` possui uma versão para feature `db-test`, o resultado será sempre um user válido. Assim, vamos ao primeiro teste:

```rust
#[cfg(test)]
mod middleware {
    use actix_web::{test, App, http::StatusCode};
    use dotenv::dotenv;
    use todo_server::todo_api_web::model::http::Clients;
    use todo_server::todo_api_web::routes::app_routes;

    use crate::helpers::read_json;

    #[actix_rt::test]
    async fn bad_request_todo_post() {
        dotenv().ok();
        let mut app =
            test::init_service(
                App::new()
                .data(Clients::new())
                .wrap(todo_server::todo_api_web::middleware::Authentication)
                .configure(app_routes)
            ).await;

        let req = test::TestRequest::post()
            .uri("/api/create")
            .header("Content-Type", "application/json")
            .set_payload(read_json("post_todo.json").as_bytes().to_owned())
            .to_request();

        let resp = test::call_service(&mut app, req).await;
        assert_eq!(resp.status(), StatusCode::BAD_REQUEST);
    }
}
```

Este teste é práticamente igual ao teste que criamos uma `todo_card`, porém possui a função `.wrap(todo_server::todo_api_web::middleware::Authentication)` associada a `App` e em vez de validar a resposta valida o status como `BAD_REQUEST`. Depois disso, podemos criar um teste que adiciona um header `x-auth` com um Jwt contendo valores aleatórios para nossos campos:

```
{
  "id": "7562bf53-6156-433b-a201-90bbc74b0127",
  "email": "my@email.com",
  "expires_at": "2014-11-28T12:00:09"
}

Algoritmo HS256 com chave `your-256-bit-==secret`

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6Ijc1NjJiZjUzLTYxNTYtNDMzYi1hMjAxLTkwYmJjNzRiMDEyNyIsImVtYWlsIjoibXlAZW1haWwuY29tIiwiZXhwaXJlc19hdCI6IjMwMjAtMTEtMjhUMTI6MDA6MDkifQ.hom6KvmmLIuu3dLCSUrOK9KBWyUb0fvdX4hIay52UIY
```

O teste para estes valores é:

```rust
#[actix_rt::test]
async fn good_token_todo_post() {
    dotenv().ok();
    let mut app =
        test::init_service(
            App::new()
            .data(Clients::new())
            .wrap(todo_server::todo_api_web::middleware::Authentication)
            .configure(app_routes)
        ).await;

    let req = test::TestRequest::post()
        .uri("/api/create")
        .header("Content-Type", "application/json")
        .header("x-auth", "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6Ijc1NjJiZjUzLTYxNTYtNDMzYi1hMjAxLTkwYmJjNzRiMDEyNyIsImVtYWlsIjoibXlAZW1haWwuY29tIiwiZXhwaXJlc19hdCI6IjMwMjAtMTEtMjhUMTI6MDA6MDkifQ.hom6KvmmLIuu3dLCSUrOK9KBWyUb0fvdX4hIay52UIY")
        .set_payload(read_json("post_todo.json").as_bytes().to_owned())
        .to_request();

    let resp = test::call_service(&mut app, req).await;
    println!("{:?}", resp);
    assert_eq!(resp.status(), StatusCode::CREATED);
}
```

Se executarmos este teste vamos receber como resposta um `panic!`, pois o middleware vai conter um `is_active` false que vai se encadear para um `Err`. Assim, precisaremos fazer algumas modificações em `scan_user`, `User.from` e `handle` de JwtValue. Com isso, as modificações serão em ordem de encadeamento:

```rust
// src/todo_api/model/core.rs
impl Handler<JwtValue> for DbExecutor {
    type Result = bool;

    #[cfg(not(feature = "dbtest"))]
    fn handle(&mut self, msg: JwtValue, _: &mut Self::Context) -> Self::Result {
        use crate::todo_api::db::auth::scan_user;

        let user = scan_user(String::from(&msg.email), &self.0.get().expect("Failed to open connection"));
        match user {
            Err(_) => false,
            Ok(user) => {
                match (user.is_active, validate_jwt_date(user.expires_at), user.id.to_string() == msg.id) {
                    (true, true, true) => true,
                    _ => false
                }
            }
        }
    }

    #[cfg(feature = "dbtest")]
    fn handle(&mut self, msg: JwtValue, _: &mut Self::Context) -> Self::Result {
        use crate::todo_api::db::auth::test_scan_user;

        let user = test_scan_user(String::from(&msg.email), String::from(&msg.id), &self.0.get().expect("Failed to open connection"));
        match user {
            Err(_) => false,
            Ok(user) => {
                match (user.is_active, validate_jwt_date(user.expires_at), user.id.to_string() == msg.id) {
                    (true, true, true) => true,
                    _ => false
                }
            }
        }
    }
}

// src/todo_api/db/auth.rs
#[cfg(not(feature = "dbtest"))]
pub fn scan_user(user_email: String, conn: &PgConnection) -> Result<User, DbError> {
    use crate::schema::auth_user::dsl::*;

    let items = auth_user.filter(email.eq(&user_email)).load::<User>(conn);

    match items {
        Ok(users) if users.len() > 1 => Err(DbError::DatabaseConflit),
        Ok(users) if users.len() < 1 => Err(DbError::CannotFindUser),
        Ok(users) => Ok(users.first().unwrap().clone().to_owned()),
        Err(_) => Err(DbError::CannotFindUser),
    }
}

#[cfg(feature = "dbtest")]
pub fn scan_user(user_email: String, _conn: &PgConnection) -> Result<User, DbError> {
    use crate::schema::auth_user::dsl::*;
    use diesel::debug_query;
    use diesel::pg::Pg;
    let query = auth_user.filter(email.eq(&user_email));
    let expected = "SELECT \"auth_user\".\"email\", \"auth_user\".\"id\", \"auth_user\".\"password\", \"auth_user\".\"expires_at\", \"auth_user\".\"is_active\" FROM \"auth_user\" WHERE \"auth_user\".\"email\" = $1 -- binds: [\"my@email.com\"]".to_string();

    assert_eq!(debug_query::<Pg, _>(&query).to_string(), expected);
    Ok(User::from(user_email, "this is a hash".to_string()))
}

#[cfg(feature = "dbtest")]
pub fn test_scan_user(user_email: String, auth_id: String, _conn: &PgConnection) -> Result<User, DbError> {
    Ok(User::test_from(user_email, "this is a hash".to_string(), auth_id))
}

// src/todo_api/model/auth.rs
impl User {
    pub fn from(email: String, password: String) -> Self {
        let utc = crate::todo_api::db::helpers::one_day_from_now();

        Self {
            email: email,
            id: uuid::Uuid::new_v4(),
            password: password,
            expires_at: utc.naive_utc(),
            is_active: false,
        }
    }

    #[cfg(feature = "dbtest")]
    pub fn test_from(email: String, password: String, id: String) -> Self {
        let utc = crate::todo_api::db::helpers::one_day_from_now();

        Self {
            email: email,
            id: uuid::Uuid::parse_str(&id).unwrap(),
            password: password,
            expires_at: utc.naive_utc(),
            is_active: true,
        }
    }
    // ...
}
```

Precisamos modificar `handle` pois precisamos enviar `id` como argumento para validar seu valor posteriormente. Como enviamos id, precisamos modificar `scan_user` para criar um `test_scan_user` que receba o `id` como argumento e passe para um `User::from` que também suporte configurar `id` e definir `is_active` como `true`. Com isso, todos nossos testes passam e podemos prosseguir para os últimos passos, criar um CI, obter um `todo` pelo seu `id` e fazer update de um `todo`.

[Anterior](05-auth.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](07-ci.md)