[Anterior](04-serving.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](06-middleware.md)

# Autenticação

Criaremos funções do serviço para registrar e para fazer login no nosso serviço. Além disso, implementaremos um middleware que protege nossos endpoints de usuários não autenticados. Para realizar isso vamos utilizar a crate `Diesel` para lidar com a base de dados, que será o Postgres. Para isso precisamos seguir alguns passos:

1. Instale a `diesel_cli`, pois este binário ajuda a gerenciar o projeto. Utilize `cargo install diesel_cli` para isso. Para compilar o `diesel_cli` é preciso ter a lib `lmysqlclient`, no MacOS podemos fazer isso com `brew install mysql` ou simplesmente utilizar `cargo install diesel_cli --no-default-features --features postgres` para isntalar somente o conector de `postgres`.
2. Ter um container disponível `docker run -i --rm --name auth-db -p 5432:5432 -e POSTGRES_USER=auth -e POSTGRES_PASSWORD=secret -d postgres`
3. Para utilizar o `diesel_cli` executamos o comando `diesel setup`, mas para isso precisamos da url do postgress em um arquivo `.env` com `echo DATABASE_URL=postgres://auth:secret@localhost/auth_db > .env`. Agora executamos `diesel setup` para estabelecer a conexão.
4. Depois podemos criar nossas migrações com `diesel migration generate create_auth`, note a pasta `migrations` com duas subpastas cada uma contendo um `up.sql` e um `down.sql`.
5. Na segunda pasta vamos criar a tabela `auth_user` em `up.sql`:

```sql
-- up.sql
CREATE TABLE auth_user (
    email VARCHAR(100) NOT NULL PRIMARY KEY,
    id UUID NOT NULL,
    password VARCHAR(64) NOT NULL, --bcrypt hash
    expires_at TIMESTAMP NOT NULL
);
```

```sql
--down.sql
DROP TABLE auth_user;
```

6. Agora basta executar as migrations com `diesel migration run`, caso você queira reverter as migrations basta executar `diesel migration redo`. Note a criação de um arquivo `src/schema.rs` em nosso projeto:

```rust
table! {
    auth_user (email) {
        email -> Varchar,
        id -> Uuid,
        password -> Varchar,
        expires_at -> Timestamp,
    }
}
```

> A macro table! gera código baseado no schema da base de dados que representes todas as tabelas e colunas.

7. Tipicamente um schema não é criado na mão e sim gerado pelo binário `diesel`. Quando executamos `diesel setup`, um arquivo `diesel.toml` é criado para indicar ao `Diesel` para manter o arquivo `src/schema.rs` por nós.

> Nota sobre Diesel em Produção
> 
> Quando em produção você talvez prefira executar suas migrações na inicialização da aplicação. Assim, a crate Diesel disponibiliza a macro `embed_migrations!`, permitindo embedar os scripts de migração como parte final do binário. Para usa-la, basta incluir `mbedded_migrations::run(&db_conn)` no início de suas `main` e as migrações serão executadas.

## Configurando o Postgress com Rust

Agora podemos começar a evoluir a autenticação do nosso servidor, para isso devemos adicionar algumas crates ao `[dependencies]` do Cargo.toml:

```toml
actix = "0.9.0"
chrono = { version = "0.4.10", features = ["serde"] }
diesel = { version = "1.4.3", features = ["postgres", "uuidv07", "r2d2", "chrono"] }
dotenv = "0.15.0"
r2d2 = "0.8.8"
```

A abordagem que vamos seguir aqui é diferente da apresentada no guia do diesel, consulte bibliografia para obter o link, pois vamos tentar tirar proveito do sistema de actors do actix (caso você queira, é um bom exercício aplicar a mesma estratégia ao `DynamoDbClient`). Assim, em nosso módulo `src/todo_api/db/helpers.rs` vamos criar uma struct `DbExecutor`, com um tipo de conexão de pool, que vai implementar a trait `Actor` do actix:

```rust
use actix::{Actor, SyncContext};
use diesel::pg::PgConnection;
use diesel::r2d2::{ConnectionManager, Pool};

pub struct DbExecutor(pub Pool<ConnectionManager<PgConnection>>);

impl Actor for DbExecutor {
    type Context = SyncContext<Self>;
}
```

> Actors
> 
>  Actors se comunicam exclusivamente pela troca de mensagens. Assim, o actor que envia a mensagem irá, opcionalmente, esperar pela respostas. Além disso, actors não são referenciados diretamente, mas sim pelos seus endereços. Qualquer tipo no RUst pode se tornar um actor, o único requerimento é que implemente a trait `Actor`.

Depois disso precisamos adicionar a struct `DbExecutor` ao nosso `Clients`, porém nosso `DbExecutor` vai precisar precisar ser envelopado em um `Addr<T>`, que corresponde ao endereço do actor:

```rust
// src/todo_api_web/model/http.rs
use actix::prelude::Addr;
use crate::todo_api::db::helpers::{client, DbExecutor};

#[derive(Clone)]
pub struct Clients {
    pub dynamo: rusoto_dynamodb::DynamoDbClient,
    pub postgres: Addr<DbExecutor>,
}

impl Clients {
    pub fn new(pg: Addr<DbExecutor>) -> Self {
        Self { 
            dynamo: client(),
            postgres: pg
        }
    }
}
```

Agora precisamos de uma função que crie o `Addr<DbExecutor>` para podemos enviar como argumento ao `new`. Essa função se chamara `db_executor_address` e estará localizada em `src/todo_api/db/helpers.rs`:

```rust
use actix::{Actor, Addr, SyncContext, SyncArbiter};
// ...
use diesel::{
    r2d2::{ConnectionManager, Pool},
    pg::PgConnection
};
use std::env;

// ...

pub struct DbExecutor(pub Pool<ConnectionManager<PgConnection>>);

impl Actor for DbExecutor {
    type Context = SyncContext<Self>;
}

pub fn db_executor_address() -> Addr<DbExecutor> {
    let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");

    let manager = ConnectionManager::<PgConnection>::new(database_url);
    let pool = r2d2::Pool::builder()
        .build(manager)
        .expect("Failed to create pool.");

    SyncArbiter::start(4, move || DbExecutor(pool.clone()))
}
```

Agora podemos modificar a função `Clients::new` para que não seja preciso passar `Addr<DbExecutor>` como argumento:

```rust
use crate::todo_api::db::helpers::{client, DbExecutor, db_executor_address};
use actix::prelude::Addr;

#[derive(Clone)]
pub struct Clients {
    pub dynamo: rusoto_dynamodb::DynamoDbClient,
    pub postgres: Addr<DbExecutor>,
}

impl Clients {
    pub fn new() -> Self {
        Self {
            dynamo: client(),
            postgres: db_executor_address(),
        }
    }
}
```

Note que ao executar o código obtemos uma falha, pois `DATABASE_URL` não está setada, agora precisamos utilizar as configurações do postgres para o `docker-compose`:

```yml
version: "3.7"
services:
# ...
  postgres:
    container_name: "postgres"
    image: postgres
    ports:
      - "5432:5432"
    networks:
      internal_net:
        ipv4_address: 172.21.1.15
    environment:
      - POSTGRES_USER=auth
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=auth_db
# ...
```

Para isso precisamos remover nosso `env_logger` nosso código e Cargo.toml. Além disso, a definição da variável de ambiente do log passa para o arquivo `.env`:

```
DATABASE_URL=postgres://auth:secret@172.21.1.15/auth_db
RUST_LOG=actix_web=info
```

Agora precisamos executar as migrações no docker compose, para isso vamos utilizar `embed_migrations!` migrations como falamos anteriormente. A macro `embed_migrations!` está disponível na crate `diesel_migrations = "1.4.0"`, adicione ela a seu `[dependencies]` do Cargo.toml. No módulo `src/schema.rs` adicione a linha `embed_migrations!();` depois da macro `table!`. E agora precisamos que o código execute a migração. Para isso adicionamos a função `run_migrations` em `create_table`, no módulo `src/todo_api/db/helpers.rs`:

```rust
use actix::{Actor, Addr, SyncArbiter, SyncContext};
use diesel::{
    prelude::*,
    r2d2::{ConnectionManager, Pool},
};
use diesel_migrations::run_pending_migrations;
use log::{debug, error};
// ...
use std::env;

// ...

pub fn create_table() {
    let client = client();
    let list_tables_input: ListTablesInput = Default::default();
    run_migrations();

    // ...
}

fn run_migrations() {
    let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let pg_conn = PgConnection::establish(&database_url)
        .expect(&format!("Error connecting to {}", database_url));
    match run_pending_migrations(&pg_conn) {
        Ok(_) => debug!("auth database created"),
        Err(_) => error!("auth database creation failed"),
    };
}
// ...
```

* Cuidado pois esta configuração do docker-compose pode consumir muita memória. Pode ser interessante executar um `docker system prune --volumes` caso seu docker falhe. 


## Criando o endpoint de cadastro de usuários

Nosso próximo passo é modelar o domínio de autenticação em `todo_api`, chamaremos nossa struct de `User` e incluiremos no módulo `src/todo_api/model/auth.rs`. A primeira coisa que devemos fazer é declarar `mod schema` em `main.rs` e `lib.rs`, e por motivos de agilidade utilizar `#[macro_use] extern crate diesel_migrations;` e `#[macro_use] extern crate diesel;` para disponibilizar as macros utilizads em `schema.rs`. Depois disso, podemos criar nossa struct `User`:

```rust
use crate::schema::*;

#[derive(Debug, Serialize, Deserialize, Queryable, Insertable)]
#[table_name = "auth_user"]
pub struct User {
    email: String,
    id: uuid::Uuid,
    password: String,
    expires_at: chrono::NaiveDateTime,
}
```

Utilizamos a linha `use crate::schema::*;` para disponibilizar a tabela `auth_user` neste contexto para que a "anotação" `table_name` funcione. Além disso, aplicamos as macros `Queryable, Insertable` para que possamos utilizar nossa struct com o postgres. Agora sabemos que vamos receber 2 argumentos para criar um user, que serão `email` e `password`, com isso podemos presupor que vamos precisar implementar uma função que gere um tipo `User` destes dois argumentos, algo como `fn from(email: String, password: String) -> User`. Assim, podemos implementar um teste para esta funcão. Para este teste vamos ter que adicionar a crate `regex = "1.3.4"` ao nosso `[dev-dependencies]` do Cargo.toml:

```rust
#[cfg(test)]
mod test {
    use super::*;
    use regex::Regex;

    #[test]
    fn user_is_correctly_created() {
        let user = User::from(String::from("email"), String::from("password"));
        let rx = Regex::new("[0-9]{4}-[0-1]{1}[0-9]{1}-[0-3]{1}[0-9]{1} [0-2]{1}[0-9]{1}:[0-6]{1}[0-9]{1}:[0-6]{1}[0-9]{1}").unwrap();

        assert_eq!(user.email, String::from("email"));
        assert_eq!(user.password, String::from("password"));
        assert!(uuid::Uuid::parse_str(&user.id.to_string()).is_ok());
        assert!(rx.is_match(&format!("{}", user.expires_at.format("%Y-%m-%d %H:%M:%S"))));
    }
}
```

Neste teste estamos testando se `email` e `password` são exatamente como enviamos, se o `Uuid` é gerado como `Uuid` e se o formato da data está de acordo coma  regex `"[0-9]{4}-[0-1]{1}[0-9]{1}-[0-3]{1}[0-9]{1} [0-2]{1}[0-9]{1}:[0-6]{1}[0-9]{1}:[0-6]{1}[0-9]{1}"`. Agora podemos implementar a funcão `from`:

```rust
impl User {
    pub fn from(email: String, password: String) -> Self {
        use chrono::{DateTime, Duration, Utc};

        let utc: DateTime<Utc> = Utc::now() + Duration::days(1);

        Self {
            email: email,
            id: uuid::Uuid::new_v4(),
            password: password,
            expires_at: utc.naive_utc(),
        }
    }
}
```

Note que está funcão começa com `use chrono::`, isso se deve ao fato de estas structs ainda não serem necessárias em outras partes do código. Depois disso vemos a linha `let utc: DateTime<Utc> = Utc::now() + Duration::days(1);`, que é depois inserida em `expires_at`, ela referencia a ideia de que o token vai sobreviver apenas até este período, que é deste instante até mais um dia. Podemos ainda simplificar esta função para extrair o `DateTime<Utc>` com `one_day_from_now()`:

```rust
impl User {
    pub fn from(email: String, password: String) -> Self {
        let utc = crate::todo_api::db::helpers::one_day_from_now();

        Self {
            email: email,
            id: uuid::Uuid::new_v4(),
            password: password,
            expires_at: utc.naive_utc(),
        }
    }
}
```

Agora `one_day_from_now` está definida no módulo `src/todo_api/db/helpers.rs` como:

```rust
use chrono::{DateTime, Duration, Utc};
// ...
pub fn one_day_from_now() -> DateTime<Utc> {
    Utc::now() + Duration::days(1)
}
```
### Adaptando o request para um modelo de banco de dados.

Agora precisamos de um modelo que represente o request HTTP de `signup`, porém nossos modelos de `Todo` estão todos em `src/todo_api_web/model/mod.rs`, assim devemos criar um módulo `model/todo.rs` e corrigir todas as chamadas:

```rust
// src/todo_api_web/controller/todo.rs
use crate::{
    // ...
    todo_api_web::model::{
        http::Clients,
        todo::{TodoCard, TodoIdResponse, TodoCardsResponse}
    }
};

// src/todo_api/db/todo.rs
use crate::todo_api_web::model::todo::TodoCard;

// src/todo_api/adapter/mod.rs
use crate::{
    todo_api::model::{StateDb, TaskDb, TodoCardDb},
    todo_api_web::model::todo::{State, Task, TodoCard},
};
// ...
#[cfg(test)]
mod test {
    use super::*;
    use crate::{
        todo_api::model::{StateDb, TaskDb, TodoCardDb},
        todo_api_web::model::todo::{State, Task, TodoCard},
    };
    // ...
}

// tests/helpers.rs
use todo_server::todo_api_web::model::todo::{State, Task, TodoCard};

// tests/todo_api_web/controller.rs
mod create_todo {
    use todo_server::todo_api_web::{
        model::todo::TodoIdResponse,
        routes::app_routes
    };
    // ...
}

mod read_all_todos {
    use todo_server::todo_api_web::{
        model::todo::{TodoCardsResponse},
        routes::app_routes
    };
    // ...
}
```

Com tudo corrigido, podemos criar o módulo `src/todo_api_web/model/auth.rs` com a struct `SignUp`:

```rust
#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
pub struct SignUp {
    pub email: String,
    pub password: String,
}
```

Note que ambos campos são `pub`, isso é porque vamos precisar deles no `adapter`. Felizmente não podemos guardar o `password` como texto em nosso banco de dados, para isso vamos utilizar uma crate chamada bcrypt, adicionando `bcrypt = "0.6"` ao nosso `[dependencies]` do Cargo.toml (caso você se interesse por criptografia e queira outras opções, sugiro olhar também as crates `argonautica` e `libreauth`). Faremos está conversão no módulo `src/todo_api/adapter/auth.rs`:

```rust
use crate::todo_api::model::auth::User;
use crate::todo_api_web::model::auth::SignUp;
use bcrypt::{hash, DEFAULT_COST};

pub fn signup_to_hash_user(su: SignUp) -> User {
    let hashed_pw = hash(su.password, DEFAULT_COST);
    User::from(su.email, hashed_pw.unwrap())
}
```

Agora importamos duas coisas ao escopo, a função `hash` e `DEFAULT_COST`. `bcrypt` possui 3 principais funções e um padrão de custo, que é `DEFAULT_COST` e definido como `12u32`. As funções são:

1. `hash` recebe um password do tipo genérico `P`, no nosso caso `password` do tipo `String` e um custo, no caso `DEFAULT_COST`.
2. `verify` verifica se o `password` enviado é igual a `hash`enviada.
3. `bcrypt` é similar ao `hash`, porém o segundo argumento é um `salt` do tipo `&[u8]`

Quanto ao custo, quanto maior o valor de custo, mais lento o hashing. Existe um benchmark com diferentes custos que apresenta uma relação de custo por velocidade:

* Custo = 4: test bench_cost_4       ... bench:   1,197,414 ns/iter (+/- 112,856)
* Custo = 10: test bench_cost_10      ... bench:  73,629,975 ns/iter (+/- 4,439,106)
* Custo = 12: test bench_cost_default ... bench: 319,749,671 ns/iter (+/- 29,216,326)
* Custo = 14: test bench_cost_14      ... bench: 1,185,802,788 ns/iter (+/- 37,571,986)

Creio que podemos escrever um teste simples para `signup_to_hash_user` como:

```rust
#[cfg(test)]
mod test {
    use super::*;
    use crate::todo_api_web::model::auth::SignUp;

    #[test]
    fn asser_signup_becomes_user() {
        let email = "my@email.com";
        let pass = "this Is a cr4zy p@ssw0rd";
        let signup = SignUp {
            email: String::from(email), 
            password: String::from(pass)
        };
        let user = signup_to_hash_user(signup);
        user.is_user_valid(email, pass)
    }
}
```

Este teste que criamos funciona da seguinte maneira, ele cria um `SignUp` com valores fixos e passa para a função adapter `signup_to_hash_user`, depois disso validamos que os inputs passados para `SignUp` formam um `User` válido com `.is_user_valid(email, pass)`. Agora a função `is_user_valid` é um pouco diferente do que já vimos, pois ela é uma função que compila apenas para testes com `#[cfg(test)]`, e possui `asserts` internos. Os asserts foram movidos para o arquivo de `src/todo_api/model/auth.rs` pois os campos de `User` são privados:

```rust
impl User {
    // ...

    #[cfg(test)]
    pub fn is_user_valid(self, email: &str, password: &str) {
        use bcrypt::verify;

        assert_eq!(self.email, String::from(email));
        assert!(verify(password, &self.password).unwrap());
        assert!(self.id.to_string().len() == 36);
    }
}
```

Com está função de teste podemos testar os valores internos de 1 User, comparando se o `email` interno é igual ao `email` recebido, se a string de `password` é um caso possível para `user.password` e se o `id` tem o tamanho de um `Uuid` do tipo `v4`. 

### Comunicando `SignUp` com o banco

Agora temos `SignUp` e podemos converter em `User` com a função adapter `signup_to_hash_user`, falta inserir `User` no banco de dados para podermos criar nosso endpoint de `signup`. O primeiro passo para isso é criarmos a função `insert_new_user` em `src/todo_api/db/auth.rs`:

```rust
use diesel::{PgConnection, prelude::*};

use crate::todo_api::model::auth::User;
use crate::todo_api::db::error::DbError;

pub fn insert_new_user(user: User, conn: &PgConnection) -> Result<(),DbError>{
    use crate::schema::auth_user::dsl::*;

    let new_user = diesel::insert_into(auth_user)
        .values(&user)
        .execute(conn);

    match new_user {
        Ok(_) => Ok(()),
        Err(_) => Err(DbError::UserNotCreated)
    }
}
```

Vamos explicar o que se passa neste módulo. Precisamos de `PgConenction` para disponibilizar uma conexnao a nosso `execute`, que é o executor da nossa query. `prelude::*` serve para disponivilizar funções como `execute`. Além disso, criamos um módulo para conter todos nossos erros de banco de dados em `src/todo_api/db/error.rs`, que possui o enum `DbError` que veremos a seguir. Depois disso, temos `use crate::schema::auth_user::dsl::*;`, que disponibiliza a table `auth_user` para utilizar em `insert_into(auth_user)`. Temos, também, `diesel::insert_into(auth_user).values(&user).execute(conn)` que insere na tabale `auth_user` com `insert_into`, define seus valores de inserção com `values`, recebendo a struct `User`, e executa a query com `execute`, ou com `get_result` caso você queria algum dos valores existente no banco após a inserção. Por último aplicamos um `match` ao tipo `Result` de `new_user`, caso o tipo seja `Ok` retornamos um sucesso, caso o tipo seja `Err`, retornamos o erro `DbError::UserNotCreated`. Agora vamos para a implementação da trait `Error` em `DbError`:

```rust
use std::error::Error;

#[derive(Debug)]
pub enum DbError {
    UserNotCreated
}

impl std::fmt::Display for DbError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            DbError::UserNotCreated => write!(f, "User could not be created")
        }        
    }
}

impl Error for DbError {
    fn description(&self) -> &str {
        match self {
            DbError::UserNotCreated => "User could not be created, check for possible conflits"
        } 
    }

    fn cause(&self) -> Option<&dyn Error> {
        Some(self)
    }
}
```

Por enquanto temos somente um item em `DbError`, `UserNotCreated`. E agora precisamos implementar a trait `std::error::Error` para que nosso enum possa ser utilizado como um tipo erro em nosso projeto. A trait `Error` exige duas funções `description` e `cause`. `cause` é importante caso nosso erro receba algum argumento, pois nos permite retornar coisas específica com `&dyn`, já description é o texto que veremos quando, por exemplo, logarmos o erro. Além disso, notamos que a trait `Error` exige a implementação de `std::fmt::Display`, que corresponde ao `to_string()`. Agora podemos criar uma versão de teste desta função da seguinte forma:

```rust
// src/todo_api/db/auth.rs
#[cfg(test)]
mod test {
    use diesel::debug_query;
    use diesel::pg::Pg;
    use crate::schema::auth_user::dsl::*;

    #[test]
    fn insert_user_matches_url() {
        use crate::todo_api::model::auth::User;

        let user = User::from(String::from("email@my.com"), String::from("pswd"));
        let query = diesel::insert_into(auth_user).values(&user);
        let sql = String::from("INSERT INTO \"auth_user\" (\"email\", \"id\", \"password\", \"expires_at\") VALUES ($1, $2, $3, $4) \
                -- binds: [\"email@my.com\", ") + &user.id.to_string() + ", \"pswd\", " + &format!("{:?}", user.expires_at) +"]";
        assert_eq!(sql, debug_query::<Pg, _>(&query).to_string());
    }
}
```

Eu acredito que testes que comparam strings caractere a caractere é uma péssima ideia, mas a comunidade Diesel parece gostar, quando formos testar a nível de integração usaremos outra estratégia. Notamos a presença de `debug_query` e `Pg`, ambos são responsáveis por nos permitir debugar a query que que montamos com `diesel::insert_into(auth_user).values(&user)` sem executá-la. Depois fazemos um assert de nossa query com `let sql = String::from("INSERT INTO \"auth_user\" (\"email\", \"id\", \"password\", \"expires_at\") VALUES ($1, $2, $3, $4) \ -- binds: [\"email@my.com\", ") + &user.id.to_string() + " \"pswd\", " + &format!("{:?}", user.expires_at) +"]";` que é uma string com o valor que esperamos para o sql. Note que estamos acessando os campos `id` e `expires_at` neste teste, para fazermos isso mudamos a implementação de `User` um pouco:

```rust
// src/todo_api/model/auth.rs
#[derive(Debug, Serialize, Deserialize, Queryable, Insertable)]
#[table_name = "auth_user"]
pub struct User {
    #[cfg(test)] pub email: String,
    #[cfg(not(test))] email: String,
    #[cfg(test)] pub id: uuid::Uuid,
    #[cfg(not(test))] id: uuid::Uuid,
    #[cfg(test)] pub password: String,
    #[cfg(not(test))] password: String,
    #[cfg(test)] pub expires_at: chrono::NaiveDateTime,
    #[cfg(not(test))] expires_at: chrono::NaiveDateTime,
}
// ...
```

Fizemos com que os campos sejam públicos para teste e privado para todos os outros ambientes. Agora implementaremos o endpoint em si.

> Formatando `expires_at` e a crate Chrono
>
> Muitas vezes é complicado acertar diretamente qual o formato que você quer que sua string contendo a data tenha, por isso, aqui está um bom referencial para o tipo `UTC` do Chrono:
> * `assert_eq!(dt.format("%Y-%m-%d %H:%M:%S").to_string(), "2014-11-28 12:00:09");`
> * `assert_eq!(dt.format("%a %b %e %T %Y").to_string(), "Fri Nov 28 12:00:09 2014");`
> * `assert_eq!(dt.format("%a %b %e %T %Y").to_string(), dt.format("%c").to_string());`
> * `assert_eq!(dt.to_string(), "2014-11-28 12:00:09 UTC");`
> * `assert_eq!(dt.to_rfc2822(), "Fri, 28 Nov 2014 12:00:09 +0000");`
> * `assert_eq!(dt.to_rfc3339(), "2014-11-28T12:00:09+00:00");`
> * `assert_eq!(format!("{:?}", dt), "2014-11-28T12:00:09Z");`

### Definindo o endpoint

Nosso primeiro passo é definir um teste para este endpoint e a partir deste teste podemos implementar a solução:

```rust
mod  auth {
    use actix_web::{
        test, App,
        http::StatusCode,
    };
    use actix_service::Service;
    use todo_server::todo_api_web::{
        routes::app_routes
    };
    use dotenv::dotenv;
    use crate::helpers::{read_json};
    use todo_server::todo_api_web::model::http::Clients;


    #[actix_rt::test]
    async fn signup_returns_created_status() {
        dotenv().ok();
        let mut app = test::init_service(
            App::new()
                .data(Clients::new())
                .configure(app_routes)
        ).await;
    
        let signup_req = test::TestRequest::post()
            .uri("/auth/signup")
            .header("Content-Type", "application/json")
            .set_payload(read_json("signup.json").as_bytes().to_owned())
            .to_request();

        let resp = app.call(signup_req).await.unwrap();
        assert_eq!(resp.status(), StatusCode::CREATED);
    }
}
```

Nosso teste define `signup_req` como o request que vamos enviar para `app.call(signup_req)`, mas este request possui uma nova `URI` `"/auth/signup"` e um novo arquivo Json com o conteúdo de post `"signup.json"`. Precisamos então definir este arquivo em `dev-resources` e implementar a rota. Note que o `assert` neste caso é somente para verificar se o usuário foi criado. O arquivo `signup.json` possui o seguinte formato:

```json
{
    "email": "my@email.com",
    "password": "My cr4azy p@ssw0rd"
}
```

> Reconfigurando os testes
> 
> Ao executarmos os testes agora termos um retorno de `InternalServerError`, isso se deve ao fato de que `DbExecutor` não consegue encontrar a `DATABASE_URL` que está associada ao banco. Isso se deve pelo fato de estarmos utilizando uma url diferente para o docker compose e outra para testes locais. Além disso, Postgres é mais complicado que DynamoDB no sentido de que o cliente realmente tenta estabelecer uma conexão para iniciar e para isso precisamos de uma base de dados falsa executando. Além disso, essa base deve estar migrada para as queries ocorrerem sem problemas. Assim, nosso `make test` fica mais complicado:
> 
> ```Makefile
> db:
> 	docker run -i --rm --name auth-db -p 5432:5432 -e POSTGRES_USER=auth -e POSTGRES_PASSWORD=secret -d postgres
> 
> test: db
> 	diesel setup
> 	diesel migration run
> 	cargo test --features "dbtest"
> 	diesel migration redo
>  
> clear-db:
>   docker ps -a | awk '{ print $1,$2 }' | grep postgres | awk '{print $1 }' | xargs -I {} docker stop {}
> ```
>
> Note que a partir da agora para rodar os testes precisamos de um container postgres configurado (`setup` e `migration run`) para podermos executar nossos testes sem quebrar o `DbExecutor`. Pode ser necessário adicionar um `sleep 3` depois de `test: db` para dar tempo do container executar. A última linha iniciada em `docker ps` serve para remover o container que executamos. Além disso, `DbExecutor` depende de `dotenv`, assim, devemos incluir `dotenv().ok()` antes de executar os testes e incorporar o `dotenv` no escopo com `use dotenv::dotenv;`. 

Agora que configuramos o teste, precisamos fazer a configuração de rotas, `app_routes` passa a ser:

```rust
use crate::todo_api_web::controller::{
    // ...
    auth::{signup_user}
};
use actix_web::{web, HttpResponse};

pub fn app_routes(config: &mut web::ServiceConfig) {
    config.service(
        web::scope("/")
            .service(
                // ...
            )
            .service(
                web::scope("auth/")
                    .route("signup", web::post().to(signup_user))
            )
            .route("ping", web::get().to(pong))
            .route("~/ready", web::get().to(readiness))
            .route("", web::get().to(|| HttpResponse::NotFound())),
    );
}
```

Ainda precisamos implementar o controller `signup_user` no módulo de controllers `auth`, porém ao contrário do método que vinhamos utilizando para inserir `User` no nosso banco de dados, que é o default do Diesel, vamos tirar proveito do sistema de actors do Actix e implementar um handler para permitir a comunição entre nosso serviço e o `Diesel` por mensagens. Assim, devemos mudar nossa struct `SignUp` para que ela implemente as traits `Handler` e `Message`, que vão nos permitir enviar mensagens para o `DbExecutor`:

```rust
use actix::prelude::*;
use crate::todo_api::{
    db::{
        error::DbError,
        helpers::DbExecutor,
    },
    adapter
};
#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
pub struct SignUp {
    pub email: String,
    pub password: String,
}

impl Message for SignUp {
    type Result = Result<(), DbError>;
}

impl Handler<SignUp> for DbExecutor {
    type Result = Result<(), DbError>;

    fn handle(&mut self, msg: SignUp, _: &mut Self::Context) -> Self::Result {
        use crate::schema::auth_user::dsl::*;
        use crate::diesel::RunQueryDsl;

        let user = adapter::auth::signup_to_hash_user(msg);
        let new_user = diesel::insert_into(auth_user)
            .values(&user)
            .execute(&self.0.get().expect("Failed to open connection"));

        match new_user {
            Ok(_) => Ok(()),
            Err(_) => Err(DbError::UserNotCreated)
        }
    }
}
```

Para a trait `Message` devemos implementar o tipo de retorna da comunicação, como no nosso caso não vamos retornar nada deixamos o `()` e caso ocorra um erro, retornamos o que já implementamos, `DbError`. Depois disso implementamos o `Handler` para `DbExecutor` com o tipo de mensagem `SignUp`, que possui a função `handle`. O primeiro argumento de `handle` é o prório `DbExecutor`, que está implementado como `struct DbExecutor(pub Pool<ConnectionManager<PgConnection>>)`, o segundo argumento é a mensagem, no nosso caso `SignUp`, e o terceiro argumento é o contexto do actix. Note que a função `handle` é praticamente igual a `insert_new_user`, mas em vez de passarmos um `PgConnection` passamos um `PooledConnection`, uma referência ao `Pool` de cone≈ões que criamos em `DbExecutor`, e para isso precisamos adicionar `use crate::diesel::RunQueryDsl;` que altera nosso `execute` para poder realizar erstá operação. Com isto encaminhado, agora podemos criar o controller. Este controller será um pouco diferente do que usamos usualmente, pois o adapter se encontra dentro do `Handler` e o controller simplesmente ficará responsável por fazer a comunicação via mensagem entre os actors:

```rust
use actix_web::{HttpResponse, web, Responder};
use log::{error};
use crate::{
    todo_api_web::model::{
        http::Clients,
        auth::SignUp,
    }
};

pub async fn signup_user(state: web::Data<Clients>, info: web::Json<SignUp>) -> impl Responder {
    let signup = info.into_inner();

    let resp = state.postgres
        .send(signup)
        .await;

    match resp {
        Ok(_) => HttpResponse::Created(),
        Err(e) => {
            error!("{:?}",e);
            HttpResponse::InternalServerError()
        },
    }
}
```

Note que agora estamos fazendo a conversão do tipo `web::Json<SignUp>` na nossa struct `SignUp` com `let signup = info.into_inner();` e enviando seu conteúdo para o `DbExecutor` através de:

```rust
let resp = state.postgres
    .send(signup)
    .await;
```

Se executarmos os testes agora veremos que eles falham, pois nosso teste tenta invocar o banco de dados de verdade, para isso, podemos utilizar a função que criamos anteriormente `insert_new_user` dentro do `Handler` para abstrair a lógica com o banco de dados e nos permitir utilizar features. Assim, a primeira mudança passa a ser o `handle` que utiliza o `insert_new_user`:

```rust
impl Handler<SignUp> for DbExecutor {
    type Result = Result<(), DbError>;

    fn handle(&mut self, msg: SignUp, _: &mut Self::Context) -> Self::Result {
        use crate::todo_api::db::auth::insert_new_user;

        let user = adapter::auth::signup_to_hash_user(msg);

        insert_new_user(user, &self.0.get().expect("Failed to open connection"))
    }
}
```

Agora precisamos implementar uma solução de `insert_new_user`que utilize a feature `dynamo`:

```rust
use diesel::{PgConnection, prelude::*};

use crate::todo_api::model::auth::User;
use crate::todo_api::db::error::DbError;

#[cfg(not(feature = "dynamo"))]
pub fn insert_new_user(user: User, conn: &PgConnection) -> Result<(),DbError>{
    use crate::schema::auth_user::dsl::*;

    let new_user = diesel::insert_into(auth_user)
        .values(&user)
        .execute(conn);

    match new_user {
        Ok(_) => Ok(()),
        Err(_) => Err(DbError::UserNotCreated)
    }
}

#[cfg(feature = "dynamo")]
pub fn insert_new_user(_user: User, _: &PgConnection) -> Result<(),DbError>{
    use crate::schema::auth_user::dsl::*;
    use diesel::debug_query;
    use diesel::pg::Pg;

    let user = User::from(String::from("my@email.com"), String::from("My cr4azy p@ssw0rd"));
    let query = diesel::insert_into(auth_user).values(&user);
    let sql = "INSERT INTO \"auth_user\" (\"email\", \"id\", \"password\", \"expires_at\") VALUES ($1, $2, $3, $4) \
            -- binds: [\"my@email.com\", ";
    assert!(debug_query::<Pg, _>(&query).to_string().contains(sql));
    assert!(debug_query::<Pg, _>(&query).to_string().contains("My cr4azy p@ssw0rd"));

    Ok(())
}
```

Com o `#[cfg(feature = "dynamo")]` fazemos uma query para `diesel_query` com os valores de `user` (não usamos `_user` pois seus campos são privados), como fizemos no módulo de testes e depois fazemos um assert que a query retornada de `debug_query::<Pg, _>(&query).to_string()` contém a substring `sql` e que contém a substring de password `"My cr4azy p@ssw0rd"`. Depois disso retornamos `Ok(())` para conformar com o esperado do `Result`.

## Validando email e password

Agora vamos fazer algo pequeno, pois nosso objetivo é garantir que o email é no formato válido `\w{1,}@\w{2,}.[a-z]{2,3}(.[a-z]{2,3})?` (regex significando qualquer conjunto de caracteres com mais de 1 elemento entre letras, números e `_`, seguido de `@`, repete o primeiro, seguido de ponto e 2 ou 3 caracteres de letras, seguido pela possível existência de ponto e 2 ou 3 caracteres de letras). Além disso, vamos garantir que o password contém umais de 32 caracteres, com letras maiúsculas e minúsculas, números e alguns caracterés especiais. 

No controller `signup_user` adicionaremos uma validação da string de email com a crate Regex. Para isso definiremos nossa regex com `Regex::new` e depois compararemos com `is_match`. Caso a validação falhe, retornaremos `HttpResponse::BadRequest()`:

```rust
pub async fn signup_user(state: web::Data<Clients>, info: web::Json<SignUp>) -> impl Responder {
    use regex::Regex;

    let email_regex = Regex::new("\\w{1,}@\\w{2,}.[a-z]{2,3}(.[a-z]{2,3})?$").unwrap();
    let signup = info.into_inner();
    if !email_regex.is_match(&signup.email) {
        return HttpResponse::BadRequest();
    }

    // ...
}
```

Agora usaremos uma regex que garante que a senha possua pelo menos uma letra maiúscula, pelo menos uma letra minúscula, pelo menos um número e pelo menos algum dos caracteres `@!=_#&~[]{}?/` com uma tamanho entre 32 e 64 caracteres. Essa regex será `[[a-z]+[A-Z]+[0-9]+(\s@!=_#&~\[\]\{\}\?\/)]{32,64}`. Cuidado que nosso teste deve falhar a partir de agora, para isso, modifiquei `signup.json` para:

```json
{
    "email": "my@email.com",
    "password": "My cr4azy p@ssw0rd My cr4azy p@ssw0rd"
}
```

E atualizei `db/auth` para validar este novo password:

```rust
#[cfg(feature = "dynamo")]
pub fn insert_new_user(user: User, _: &PgConnection) -> Result<(),DbError>{
    use crate::schema::auth_user::dsl::*;
    use diesel::debug_query;
    use diesel::pg::Pg;

    let user = User::from(String::from("my@email.com"), String::from("My cr4azy p@ssw0rd My cr4azy p@ssw0rd"));
    let query = diesel::insert_into(auth_user).values(&user);
    let sql = "INSERT INTO \"auth_user\" (\"email\", \"id\", \"password\", \"expires_at\") VALUES ($1, $2, $3, $4) \
            -- binds: [\"my@email.com\", ";
    assert!(debug_query::<Pg, _>(&query).to_string().contains(sql));
    assert!(debug_query::<Pg, _>(&query).to_string().contains("My cr4azy p@ssw0rd My cr4azy p@ssw0rd"));

    Ok(())
}
```

Assim, podemos implementar a mudança no controller com:

```rust
pub async fn signup_user(state: web::Data<Clients>, info: web::Json<SignUp>) -> impl Responder {
    use regex::Regex;

    let email_regex = Regex::new("\\w{1,}@\\w{2,}.[a-z]{2,3}(.[a-z]{2,3})?$").unwrap();
    let pswd_regex = Regex::new("[[a-z]+[A-Z]+[0-9]+(\\s@!=_#&~\\[\\]\\{\\}\\?\\/)]{32,64}").unwrap();
    
    let signup = info.into_inner();
    if !(email_regex.is_match(&signup.email) && pswd_regex.is_match(&signup.password)) {
        return HttpResponse::BadRequest();
    }

    // ...
}
```

Um bom exercício aqui seria criar alguns testes para o controller validar as novas regras do `email` e do `password`, lembrando que o teste de cenário válido já acontece a nível de integração. Alguns possíveis `emails` de teste são `"my_email.com.br"` ou `"my@email.com.br.us"`, além disso alguns casos interessantes de teste para `passwords` são `"My Cr4zy p@ssw0rd"`, `"my cr4zy p@ssw0rd my cr4zy p@ssw0rd"` e `"My Crazy password My Crazy password"`.

## Implementando login

O objetivo de nosso endpoint de login será retornar um token `jwt` com informações garantindo a validade do sistema. Assim, nosso endpoint receberá um `email` e um `password`, validará se o password é válido e retornará um token `jwt`, que passará a ser validado nos outros endpoints. 

Podemos agora mudar a feature `dynamo` que atua sobre o Postgres e o DynamoDB para `db-test`. Para isso, devemos adicionar a feature a nosso Cargo.toml:

```toml
[features]
db-test = []
```

E a nosso Makefile:

```sh
test: db
	sleep 2
	diesel setup
	diesel migration run
	cargo test --features "dbtest"
	diesel migration redo

run-local:
	cargo run --features "db-test"
# ...
```

Por último, devemos modificar o arquivo `src/todo_api/db/auth.rs` para utilizar a nova feature:

```rust
use diesel::{PgConnection, prelude::*};

use crate::todo_api::model::auth::User;
use crate::todo_api::db::error::DbError;

#[cfg(not(feature = "db-test"))]
pub fn insert_new_user(user: User, conn: &PgConnection) -> Result<(),DbError>{
    // ...
}

#[cfg(feature = "db-test")]
pub fn insert_new_user(user: User, _: &PgConnection) -> Result<(),DbError>{
    // ...

    Ok(())
}
// ...
```

Repita isso para os outros cenários.

### Criando o endpoint de login

A partir deste momento vou mudar a forma como apresento os testes, pois creio que já temos uma boa ideia de como eles funcionam. Assim, vou apresentar o teste que escrevi para cada endpoint, mas não resolverei eles mais de forma a relacionar o código sendo escrito ao teste que queremos resolver. Isso se deve ao fato de que eles são praticamente iguais. Assim, o teste deste endpoint seria apenas validar que o status é `200`, mas vamos mudar um pouco e esperar que a resposta venha com uma chave Json `token`:

```rust
mod  auth {
    use actix_web::{
        test, App,
        http::StatusCode,
    };
    use todo_server::todo_api_web::{
        routes::app_routes
    };
    use crate::helpers::{read_json};
    use dotenv::dotenv;

    // ...
    #[actix_rt::test]
    async fn login_returns_token() {
        let mut app = test::init_service(
            App::new()
                .data(Clients::new())
                .configure(app_routes)
        ).await;

        let login_req = test::TestRequest::post()
            .uri("/auth/login")
            .header("Content-Type", "application/json")
            .set_payload(read_json("signup.json").as_bytes().to_owned())
            .to_request();

        let resp_body = test::read_response(&mut app, login_req).await;

        let jwt: String = String::from_utf8(resp_body.to_vec()).unwrap();
        
        assert!(jwt.contains("token"));
    }
}
```

Nosso próximo passo será definir o endpoint `/auth/login` que receberá um `POST` com um Json representado pela struct `Login`, que contém os mesmos campos de `SignUp`. Faremos uma nova struct para podermos tirar mais proveito do sistema de actors do Actix.

```rust
use crate::todo_api_web::controller::{
    // ...
    auth::{signup_user, login}
};

pub fn app_routes(config: &mut web::ServiceConfig) {
    config.service(
        web::scope("/")
            // ...
            .service(
                web::scope("auth/")
                    .route("signup", web::post().to(signup_user))
                    .route("login", web::post().to(login))
            )
            // ...
            .route("", web::get().to(|| HttpResponse::NotFound())),
    );
}

```

Aqui adicionamo uma rota `login` que envia o request para o controller `login`. Agora vamos ao controller `login`:

```rust
pub async fn login(state: web::Data<Clients>, info: web::Json<Login>) -> impl Responder {
    use regex::Regex;

    let email_regex = Regex::new("\\w{1,}@\\w{2,}.[a-z]{2,3}(.[a-z]{2,3})?$").unwrap();
    let pswd_regex = Regex::new("[[a-z]+[A-Z]+[0-9]+(\\s@!=_#&~\\[\\]\\{\\}\\?)]{32,64}").unwrap();
    
    let login_user = info.clone();
    if !(email_regex.is_match(&login_user.email) && pswd_regex.is_match(&login_user.password)) {
        return HttpResponse::BadRequest().finish();
    }

    let resp = state.postgres
        .send(login_user)
        .await;

    match resp {
        Err(e)  => {
            error!("{:?}",e);
            HttpResponse::NoContent().finish()
        },
        Ok(r_users) => {
            match r_users {
                Err(e) => {
                    error!("{:?}",e);
                    HttpResponse::NoContent().finish()
                },
                Ok(users) => {
                    let user = users.first().unwrap();
                    match user.verify(info.clone().password) {
                        Ok(true) => generate_jwt(user, state).await,
                        Ok(false) => HttpResponse::NoContent().finish(),
                        Err(_) => HttpResponse::NoContent().finish()

                    }
                }
            }
        },
    }
}
```

A primeira coisa que podemos notar no controller de `login` é o `is_match` das regex, lembrando que usar regex pode sempre ser algo perigoso e devemos ter muito cuidado. Isso é algo que claramente podemos extrair. Em seguida repetimos o processo de outros outros controllers e enviamos uma mensagem com `Login` em `state.postgres.send(login_user).await`, nesta chamada recebemos um vetor de `User` que passam nosso filtro, porém como estamos filtrando pela chave primária `email` não pode haver conflitos. creio que a estração das verificações de email e de senha por regex fica com a seguinte cara:

```rust
pub async fn signup_user(state: web::Data<Clients>, info: web::Json<SignUp>) -> impl Responder {
    let signup = info.into_inner();
    if !is_email_pswd_valids(&signup.email, &signup.password) {
        return HttpResponse::BadRequest();
    }

    // ...
}

pub async fn login(state: web::Data<Clients>, info: web::Json<Login>) -> impl Responder {
    let login_user = info.clone();
    if !is_email_pswd_valids(&login_user.email, &login_user.password) {
        return HttpResponse::BadRequest().finish();
    }

    // ...
}

pub fn is_email_pswd_valids(email: &str, pswd: &str) -> bool {
    use regex::Regex;

    let email_regex = Regex::new("\\w{1,}@\\w{2,}.[a-z]{2,3}(.[a-z]{2,3})?$").unwrap();
    let pswd_regex = Regex::new("[[a-z]+[A-Z]+[0-9]+(\\s@!=_#&~\\[\\]\\{\\}\\?)]{32,64}").unwrap();
    
    email_regex.is_match(email) && pswd_regex.is_match(pswd)
}
```

A vantagem deste formato, é que executar os testes fica ainda mais fácil, pois passam a ser validações unitárias, e o motivo pelo qual deixei anteriormente como exercícios. Assim, os testes podem ser como a seguir:

```rust
#[cfg(test)]
mod valid_email_pswd {
    use super::is_email_pswd_valids;

    #[test]
    fn valid_email_and_pswd() {
        assert!(is_email_pswd_valids("my@email.com", "My cr4zy P@ssw0rd My cr4zy P@ssw0rd"));
    }

    #[test]
    fn invalid_emails() {
        assert!(!is_email_pswd_valids("my_email.com", "My cr4zy P@ssw0rd My cr4zy P@ssw0rd"));
        assert!(!is_email_pswd_valids("my@email.com.br.us", "My cr4zy P@ssw0rd My cr4zy P@ssw0rd"));
    }

    #[test]
    fn invalid_passwords() {
        assert!(!is_email_pswd_valids("my@email.com.br", "My cr4zy P@ssw0rd"));
        assert!(is_email_pswd_valids("my@email.com", "my cr4zy p@ssw0rd my cr4zy p@ssw0rd"));
        assert!(is_email_pswd_valids("my@email.com", "My crazy P@ssword My crazy P@ssword"));
        assert!(is_email_pswd_valids("my@email.com", "My cr4zy Passw0rd My cr4zy Passw0rd"));
    }
}
```

Também podemos observar que no controller `login` há uma série de `HttpResponse::NoContent().finish()` para qualquer caso de erro. Duas coisas para observarmos aqui, a primeira é a presença de `finish` que se deve ao método `generate_jwt` que retorna um `HttpResponse`, a segunda é que presumo que quando alguém tenta logar em um serviço e ocorro qualquer problema, o serviço deve responder um `2XX` sem nenhuma informação, por isso do `NoContent`.

Agora podemos seguir para o caso que todas as extrações de `resp` via pattern matching e chegar em `user.verify(info.clone().password)`. O objetivo de função é validar que o `password` de `info: web:Json<Login>` é um password possível para a hash de `user.password`. Como está função é somente uma camada em volta da função original, já testada, não é imprescindível implementar testes:

```rust
// src/todo_api/model/auth.rs
impl User {
    use bcrypt::{verify, BcryptResult};
    // ...

    pub fn verify(&self, pswd: String) -> BcryptResult<bool> {
        verify(pswd,&self.password)
    }
}
```

Note que a resposta de `verify` é do tipo `BcryptResult`, ou seja, temos 3 cenários:

1. `Ok(true)` -> Caso na qual o `password` enviado é uma hash válida. 
2. `Ok(false)` -> Caso na qual o `password` não é válido.
3. `Err` -> Ocorreu algum erro de validação.

O único dos casos que é importante para nós é o caso `1`, por isso é o caso que aplicamos a função `generate_jwt`, cujo objetivo será gerar um token `jwt`. Além disso, está função não funcionará para o teste que criamos pois não estamos utilizando uma hash real, assim uma solução para isso é simplesmente responder um tipo `BcryptResult<bool>` com conteúdo `true`:

```rust
impl User {
    // ...

    #[cfg(not(feature = "dbtest"))]
    pub fn verify(&self, pswd: String) -> BcryptResult<bool> {
        verify(pswd,&self.password)
    }

    #[cfg(feature = "dbtest")]
    pub fn verify(&self, pswd: String) -> BcryptResult<bool> {
        BcryptResult::Ok(true)
    }
    // ...
}
```


Antes de continuar com `generate_jwt` precisamos explorar a implementacnao de `Login`, pois é o `Login` que é afetado pela função `state.postgres.send(login_user).await`:

```rust
#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
pub struct Login {
    pub email: String,
    pub password: String,
}

impl Message for Login {
    type Result = Result<Vec<User>, DbError>;
}

impl Handler<Login> for DbExecutor {
    type Result = Result<Vec<User>, DbError>;

    fn handle(&mut self, msg: Login, _: &mut Self::Context) -> Self::Result {
        use crate::todo_api::db::auth::scan_user;

        scan_user(msg.email, &self.0.get().expect("Failed to open connection"))
    }
}
```

Login recebe um `email` e um `password` para depois procurar no banco de dados com `scan_user`:

```rust
pub fn scan_user(user_email: String, conn: &PgConnection) -> Result<Vec<User>,DbError>{
    use crate::schema::auth_user::dsl::*;

    let items = auth_user
            .filter(email.eq(&user_email))
            .load::<User>(conn);

    match items {
        Ok(users) if users.len() > 1 => Err(DbError::DatabaseConflit),
        Ok(users) if users.len() < 1 => Err(DbError::CannotFindUser),
        Ok(users) => Ok(users),
        Err(_) => Err(DbError::CannotFindUser)
    }
}
```

É bastante simples o que acontece aqui, filtramos na tabela `auth_user` por um `user_email` que seja igual ao que enviamos. Caso essa lista seja maior que 1, houve um problema no banco de dados, pois como `email` é uma chave primária não podem haver 2, ou mais, repetidos. Qualquer outro `Err` é um `DbError` de não encontrar o usuário ou problemas de conexão. Temos um `Ok` extra que valida se a lista é zero, e retorna o erro `CannotFindUser` como a cláusula `Err`. E o `Ok` restante é o caso que procuramos. Note que ainda temos um refactor a fazer aqui, este refactor é modificar o tipo de retorno `Result<Vec<User>,DbError>` para `Result<User,DbError>` utilizando um `.first().unwrap()`, já que temos certeza que esse `first` existe. Além disso, precisamos adaptar este código para o teste, já que a ação `user.filter(email.eq(&user_email)).load::<User>(conn)` não deve existir. Fazemos essa adaptação retornando um `Ok` com um `User` contendo o email que enviamos. Na função `scan_user` com a feature `dbtest` ainda fazemos um assert na query que será gerada por `auth_user.filter(email.eq(&user_email))` e validamos com o `debug_query`. Caso você prefira substituir o `password` por uma hash válida para a senha sendo enviada no teste, não seria mais necessário utilizar a `cfg feature` para `verify`:

```rust
#[cfg(not(feature = "dbtest"))]
pub fn scan_user(user_email: String, conn: &PgConnection) -> Result<User, DbError>{
    use crate::schema::auth_user::dsl::*;

    let items = auth_user
            .filter(email.eq(&user_email))
            .load::<User>(conn);

    match items {
        Ok(users) if users.len() > 1 => Err(DbError::DatabaseConflit),
        Ok(users) if users.len() < 1 => Err(DbError::CannotFindUser),
        Ok(users) => Ok(users.first().unwrap().clone().to_owned()),
        Err(_) => Err(DbError::CannotFindUser)
    }
}

#[cfg(feature = "dbtest")]
pub fn scan_user(user_email: String, _conn: &PgConnection) -> Result<User, DbError>{
    use crate::schema::auth_user::dsl::*;
    use diesel::debug_query;
    use diesel::pg::Pg;
    let query = auth_user.filter(email.eq(&user_email));
    let expected = "SELECT \"auth_user\".\"email\", \"auth_user\".\"id\", \"auth_user\".\"password\", \"auth_user\".\"expires_at\", \"auth_user\".\"is_active\" FROM \"auth_user\" WHERE \"auth_user\".\"email\" = $1 -- binds: [\"my@email.com\"]".to_string();

    assert_eq!(debug_query::<Pg, _>(&query).to_string(), expected);
    Ok(User::from(user_email, "this is a hash".to_string()))
}
```

Agora nossa `resp` de `state.postgres.send(login_user).await` pode ser resolvida em uma `match`, na qual a cláusula de `Err` vai simplesmente retornar um `NoContent` a cláusula `Ok` vai aplicar um novo `match` em `verify`. De acordo com a resposta de verify, criamos o token. O caso `Err` é simplesmente um `NoContent` porque houve um problema na criação da hash, já o caso `Ok(false)` corresponde a senha incorreta. No caso `Ok(true)`, criamos o token em `generate_jwt`:

```rust
// src/todo_api_web/controller/auth.rs
pub async fn login(state: web::Data<Clients>, info: web::Json<Login>) -> impl Responder {
    // ...

    match resp {
        Err(e)  => {
            error!("{:?}",e);
            HttpResponse::NoContent().finish()
        },
        Ok(user) => {
            let usr = user.unwrap();
            match usr.verify(info.clone().password) {
                Ok(true) => generate_jwt(usr, state).await,
                Ok(false) => HttpResponse::NoContent().finish(),
                Err(_) => HttpResponse::NoContent().finish()
            }
        }
    }
}

// src/todo_api_web/model/auth.rs
impl Message for Login {
    type Result = Result<User, DbError>;
}

impl Handler<Login> for DbExecutor {
    type Result = Result<User, DbError>;

    fn handle(&mut self, msg: Login, _: &mut Self::Context) -> Self::Result {
        use crate::todo_api::db::auth::scan_user;

        scan_user(msg.email, &self.0.get().expect("Failed to open connection"))
    }
}
```

Agora falta implementarmos o `generate_jwt` para completarmos esse fluxo.

### Gerando um token `JWT`

O objetivo de criarmos um token `JWT` é permitir que o usuário faça requisições para páginas que exigem autentição, e até autorização (é possível passar tokens de autorização), com um token de autenticação no header do request. Essa autenticação vai conter algumas informações cruciais que vão nos permitir validar este token no nosso banco de dados. As informação que vamos adicionar ao token neste momento são as contidas na struct `User` exceto `password`. 

* É importante lembrar que o tópico de segurança é bastante complicado e não é o foco do livro, assim, a solução que vamos apresentar é útil, mas longe de ser uma solução aplicável em produção. 

Infelizmente, a função `generate_jwt` é cheio de efeitos colaterais e muito dificil de testar unitariamente, assim, vamos pular os testes dele por hora. Vamos manter essa função em um módulo `core`, a ideia desse módulo é conter a lógica associada à `src/todo_api`, mesmo que a função `generate_jwt` possua muitos efeitos colaterais e estará localizada em `src/todo_api/core/mod.rs`. O primeiro Efeito colaterial dele é criar uma nova data de expiração para daqui um dia com `crate::todo_api::db::helpers::one_day_from_now().naive_utc()`. Essa data será usada para criar uma struct que fará a atualização da data em `User`. Essa struct é chamada `UpdateDate` e contém dois campos `email` e `expires_at`.

```rust
pub async fn generate_jwt(user: User, state: web::Data<Clients>) -> HttpResponse {
    let utc = crate::todo_api::db::helpers::one_day_from_now().naive_utc();

    let update_date = UpdateDate {
        email: user.email.clone(),
        expires_at: utc,
    };
    // ...
}
```

> **JWT**
>
> JWT, ou Json Web Token, é um padrão aberto baseado na **RFC 7519** que define uma forma compacta e auto contida de transmitir de forma segura entre duas partes em um formato Json. Este token pode ser assinado com uma chave secreta via `HMAC` ou chaves publicas/privadas via ` RSA` ou `ECDSA.` Estes tokens podem ser encriptados ou não e os dois principais casos de uso são autorização e troca de informações. A estrutura de um JWT é `header`, `payload` e `assinatura`, assim o formato acaba sendo algo como `hhhhh.pppppp.aaaaa`. Usualmente o `header` possui duas partes o tipo, usualmente `"typ": "jwt"` e o algoritmo que pode ser `HMAC SHA256 ou RSA`, algo como `"alg": "HS256"`. `payload` é onde as informações que queremos trocar estão armazenadas. E assinatura, ou `signature`, é uma informação de como entender esses dados. Com o algoritmo `HMAC SHA256` a criação de um JWT o seguinte formato `HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)`, note que `payload` e `header` estão em um formato `base64`.

Agora precisamos implementar a struct `UpdateDate`. Essa struct está estritamente associada a ao módulo `core` atuando somente como um complemento a lógica, por isso adicionel ela em `src/todo_api/core/model.rs`, mas se você achar mais adequado é correto também deixar `generate_jwt` em `src/todo_api/controller/core.rs` e `UpdateDate` em `src/todo_api/model/core.rs`. Agora, nossa struct também precisa poder se comunicar por mensagem com nosso postgres e para isso vamos implementar as traits `Message` e `Handle`:

```rust
use actix::prelude::*;
use crate::todo_api::{
    db::{
        error::DbError,
        helpers::DbExecutor,
    },
};

#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
pub struct UpdateDate {
    pub email: String,
    pub expires_at: chrono::NaiveDateTime,
}

impl Message for UpdateDate {
    type Result = Result<(), DbError>;
}

impl Handler<UpdateDate> for DbExecutor {
    type Result = Result<(), DbError>;

    fn handle(&mut self, msg: UpdateDate, _: &mut Self::Context) -> Self::Result {
        use crate::todo_api::db::auth::update_user_jwt_date;

        update_user_jwt_date(msg, &self.0.get().expect("Failed to open connection"))
    }
}
```

Note que o tipo `Result` da nossa `Message` é apenas um `Reuslt` com um `Ok` vazio e um erro do tipo `DbError`, `Result<(), DbError>`. Neste caso precisamos somente saber se o update da data foi bem sucedido ou falho, com qual erro. Assim, a função `handle`simplesmente atualiza a `expires_at` no banco conforme a chave `email`. É importante também garantir que `expires_at` seja do tipo `chrono::NaiveDateTime` para não termos problemas com o tipo da tabela `auth_user`. Vamos agora olhar a função `update_user_jwt_date`.

```rust
pub fn update_user_jwt_date(update_date: UpdateDate, conn: &PgConnection) -> Result<(), DbError>{
    use crate::schema::auth_user::dsl::*;

    let target = auth_user.filter(email.eq(update_date.email));
    match diesel::update(target).set(expires_at.eq(update_date.expires_at)).execute(conn) {
        Ok(_) => Ok(()),
        Err(_) => Err(DbError::TryAgain)
    }
}
```

Para `update_user_jwt_date` precisamos disponibilizar a `dsl` de `auth_user` para fazermos operações na tabela e fazemos isso com `use crate::schema::auth_user::dsl::*;`. A primeira linha de código é encontrar o `User` alvo através de um `filter` que procura a igualdade entre os campos `email` com `auth_user.filter(email.eq(update_date.email))` sendo definido em um `let target`. Depois disso fazemos um `update` nesse `target` com `diesel::update(target)` e com isso podemos fazer um `set` do campo `expires_at` com o valor de `expires_at` de `update_date` com `set(expires_at.eq(update_date.expires_at))`. O resultado disso será um tipo `Result<(), DbError>`, que podemos utilizar em um `match` para fazer pattern matching e retornar se o update foi bem sucedido. Para realizar o teste pulamos a parte do `target`e do `match`, retornando apenas um `Ok(())`:

```rust
#[cfg(not(feature = "dbtest"))]
pub fn update_user_jwt_date(update_date: UpdateDate, conn: &PgConnection) -> Result<(), DbError>{
    use crate::schema::auth_user::dsl::*;

    let target = auth_user.filter(email.eq(update_date.email));
    match diesel::update(target)
        .set((expires_at.eq(update_date.expires_at), is_active.eq(update_date.is_active)))
        .execute(conn) {
        Ok(_) => Ok(()),
        Err(_) => Err(DbError::TryAgain)
    }
}

#[cfg(feature = "dbtest")]
pub fn update_user_jwt_date(_update_date: UpdateDate, _conn: &PgConnection) -> Result<(), DbError>{
    Ok(())
}
```

De volta a `generate_jwt` fazemos este update de forma a utilizar os recursos de `Actor` enviando uma mensagem para `UpdateDate` com `let resp = state.postgres.send(update_date);`. Note que esta função é `async` e não estamos esperando ela com o `await`, isso se deve ao fato de que as duas próximas tarefas não precisam que `resp` esteja concluída. Enquanto esperamos o momento oportuno para concretizar `resp` com `await` iniciamos a criação efeitva do token e sua preparação para o tipo de resposta `Jwt`:

```rust
pub async fn generate_jwt(user: User, state: web::Data<Clients>) -> HttpResponse {
    let utc = crate::todo_api::db::helpers::one_day_from_now().naive_utc();

    let update_date = UpdateDate {
        email: user.email.clone(),
        expires_at: utc,
    };

    let resp = state.postgres
        .send(update_date.clone());

    let token_jwt = create_token(user, update_date);
    let jwt = crate::todo_api::core::model::Jwt::new(token_jwt);    

    match resp.await {
        // ...
    }
}
```

E o tipo `Jwt` localizado em `src/todo_api/core/model.rs`:

```rust
#[derive(Serialize, Deserialize, Debug)]
pub struct Jwt{
    token: String
}

impl Jwt {
    pub fn new(jwt: String) -> Self {
        Self {
            token: jwt
        }
    }
}
```

Antes de concretizarmos `resp` com um `await` criamos o token com `create_token` e passamos este valor para a struct `Jwt` com `let jwt = crate::todo_api::core::model::Jwt::new(token_jwt);`. `create_token` é a função responsável por montar o token com os campos necessários.

```rust
pub fn create_token(user: User, update_date: UpdateDate) -> String {
    use serde_json::json;
    use jsonwebtokens::{Algorithm, AlgorithmID, encode};
    use chrono::Utc;

    let alg = Algorithm::new_hmac(AlgorithmID::HS256, "secret").unwrap();
    let header = json!({ "alg": alg.name(), "typ": "jwt", "date":  Utc::now().to_string()});
    let payload = json!({ "id": user.clone().get_id(), "email": user.email, "expires_at": update_date.expires_at });
    encode(&header, &payload, &alg).unwrap()
}
```

A primeira coisa que fazemos em `create_token` é gerar o algoritmo com a struct `Algorithm`.  Como vamos utilizar um algoritmo `HMAC SHA256` chamamos a função `new_hmac` e passamos como argumento o id que tipo que vamos utilizar com `AlgorithmID::HS256` e o segredo que vai ser passado. Uma boa alternativa para não ter o segredo exposto assim é ler ele de uma variável de ambiente. Depois disso, definimos o `header` com o algoritmo, o tipo e a data de criação em `json!({ "alg": alg.name(), "typ": "jwt", "date":  Utc::now().to_string()});`, note o uso da macro `json!` vinda de `serde_json`. Da mesma forma que com header, criamos o `payload` com os campos que nos interessam, `id, email, expires_at`. Por último geramos o token passando todas estas informações como argumento para função `encode` em `encode(&header, &payload, &alg).unwrap()`.

Para finalizar precisamos que `generate_jwt` responda um status com o conteúdo do token. Para isso fazemos um match em `resp` e retornamos `HttpResponse::InternalServerError().finish()` para o caso de `Err` e para o caso de `Ok` retornamos um `HttpResponse::Ok()` com um Json contendo a struct `Jst` serializada:

```rust
pub async fn generate_jwt(user: User, state: web::Data<Clients>) -> HttpResponse {
    // ...
    let resp = state.postgres.send(update_date.clone());
    let token_jwt = create_token(user, update_date);
    let jwt = crate::todo_api::model::core::Jwt::new(token_jwt);

    match resp.await {
        Ok(_) => {
            HttpResponse::Ok()
                .content_type("application/json")
                .json(jwt)
        }
        Err(e) => {
            error!("{:?}", e);
            HttpResponse::InternalServerError().finish()
        }
    }
}
```

Login pronto. Agora precisamos implementar o logout.

## Implementando o logout

Um login é útil, mas pode ser necessário apagarmos a sessão que temos com o serviço e para fazer isso é necessário realizar um `logout`, que atende pelo método `DELETE`. Nosso logout vai modificar nosso user de modo que tenhamos um campo booleano `is_active`. Este campo tem como responsabilidade dizer se o `user` enviado no token ainda está autenticado. Assim, vamos adicionar o campo `is_active` ao struct `User`:

```rust
// src/todo_api/model/auth.rs
// ...
#[derive(Debug, Serialize, Deserialize, Clone, Queryable, Insertable)]
#[table_name = "auth_user"]
pub struct User {
    pub email: String,
    pub id: uuid::Uuid,
    #[cfg(test)] pub password: String,
    #[cfg(not(test))] password: String,
    #[cfg(test)] pub expires_at: chrono::NaiveDateTime,
    #[cfg(not(test))] expires_at: chrono::NaiveDateTime,
    pub is_active: bool
}

impl User {
    pub fn from(email: String, password: String) -> Self {
        let utc = crate::todo_api::db::helpers::one_day_from_now();

        Self {
            email: email,
            id: uuid::Uuid::new_v4(),
            password: password,
            expires_at: utc.naive_utc(),
            is_active: false
        }
    }
    // ...
}
```

Se observarmos o `rls` do editor vamos perceber o aviso de que `Insertable` está com problemas, este problema é que `is_active` não está mapeado. Para isso devemos criar uma migração com este campo, chamaremos ela de `valid_auth` e executaremos `diesel migration generate valid_auth` que criará uma nova pasta dentro de migrations, algo como `2020-02-22-011512_valid_auth`. Depois disso adicionaremos um `up.sql` e um `down.sql`:

```sql
<-- UP.sql -->
ALTER TABLE auth_user
  ADD is_active BOOLEAN NOT NULL DEFAULT 'f';

<-- DOWN.sql -->
ALTER TABLE auth_user
  DROP is_active;
```

Esse script consiste em alterar a tabela `auth_user` para conter ou não o campo `is_active`. Com isso pronto executaremos `make db` e em seguida `diesel setup` para modificar o `schema.rs` que ficará assim:

```rust
table! {
    auth_user (email) {
        email -> Varchar,
        id -> Uuid,
        password -> Varchar,
        expires_at -> Timestamp,
        is_active -> Bool,
    }
}
```

Lembre de adicionar `embed_migrations!();` depois de `table!(...)`. Antes de continuarmos com `logout` precisamos que o `login` ative a o campo `is_active` e para isso a struct `UpdateDate` precisa receber um novo campo booleano `is_active`:

```rust
// src/todo_api/core/model.rs
// ...
#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
pub struct UpdateDate {
    pub email: String,
    pub expires_at: chrono::NaiveDateTime,
    pub is_active: bool
}
// ...
```

Se rodarmos os testes agora veremos que o teste `insert_user_matches_url` de `src/todo_api/db/auth` falha pois não espera o campo `is_active`:

```
esperado: "INSERT INTO \"auth_user\" (\"email\", \"id\", \"password\", \"expires_at\") VALUES ($1, $2, $3, $4) -- binds: [\"email@my.com\", 0f3d625b-c85c-490c-b979-f20cbbd6a71d, \"pswd\", 2020-02-23T19:31:34.896595]"`,
encontrado: "INSERT INTO \"auth_user\" (\"email\", \"id\", \"password\", \"expires_at\", \"is_active\") VALUES ($1, $2, $3, $4, $5) -- binds: [\"email@my.com\", 0f3d625b-c85c-490c-b979-f20cbbd6a71d, \"pswd\", 2020-02-23T19:31:34.896595, false]"`
```

Assim, devemos editar o teste para conter o campo `is_active` com valor default `false`:

```rust
fn insert_user_matches_url() {
    use crate::todo_api::model::auth::User;

    let user = User::from(String::from("email@my.com"), String::from("pswd"));
    let query = diesel::insert_into(auth_user).values(&user);
    let sql = String::from("INSERT INTO \"auth_user\" (\"email\", \"id\", \"password\", \"expires_at\", \"is_active\") VALUES ($1, $2, $3, $4, $5) \
            -- binds: [\"email@my.com\", ") + &user.id.to_string() + ", \"pswd\", " + &format!("{:?}", user.expires_at) +", false]";
    assert_eq!(&sql, &debug_query::<Pg, _>(&query).to_string());
}
```

Agora precisamos modificar também `core/mod.rs` para setar o campo `is_active` como true na função `generate_jwt`:

```rust
pub async fn generate_jwt(user: User, state: web::Data<Clients>) -> HttpResponse {
    let utc = crate::todo_api::db::helpers::one_day_from_now().naive_utc();

    let update_date = UpdateDate {
        email: user.email.clone(),
        expires_at: utc,
        is_active: true,
    };
    // ...
}
```

Assim como a função de `db/auth` `update_user_jwt_date`, que agora precisa setar o campo `is_active` como `true`:

```rust
pub fn update_user_jwt_date(update_date: UpdateDate, conn: &PgConnection) -> Result<(), DbError>{
    use crate::schema::auth_user::dsl::*;

    let target = auth_user.filter(email.eq(update_date.email));
    match diesel::update(target)
        .set((expires_at.eq(update_date.expires_at), is_active.eq(update_date.is_active)))
        .execute(conn) {
        Ok(_) => Ok(()),
        Err(_) => Err(DbError::TryAgain)
    }
}
```

Note que agora `diesel::update(target)` precisa atualizar 2 campos, e para isso é precisa enviar como parâmetro uma tupla contendo os dois campos a serem atualizados `(expires_at.eq(update_date.expires_at), is_active.eq(update_date.is_active))`. Com isso, podemos agora continuar com o `logout`.

Para nosso `logout` precisamos começar criando o endpoint `/auth/logout` com o método delete:

```rust
use crate::todo_api_web::controller::{
    // ...
    auth::{signup_user, login, logout}
};
use actix_web::{web, HttpResponse};

pub fn app_routes(config: &mut web::ServiceConfig) {
    config.service(
        web::scope("/")
            // ...
            .service(
                web::scope("auth/")
                    .route("signup", web::post().to(signup_user))
                    .route("login", web::post().to(login))
                    .route("logout", web::delete().to(logout)),
            )
            // ...
    );
}
```

O teste para este cenário será:

```rust
#[actix_rt::test]
async fn logout_accepted() {
    dotenv().ok();
    let mut app = test::init_service(
        App::new()
            .data(Clients::new())
            .configure(app_routes)
    ).await;

    let logout_req = test::TestRequest::delete()
        .uri("/auth/logout")
        .header("Content-Type", "application/json")
        .header("x-auth", "token")
        .set_payload(read_json("logout.json").as_bytes().to_owned())
        .to_request();

    let resp = test::call_service(&mut app,logout_req).await;
    assert_eq!(resp.status(), StatusCode::ACCEPTED);
}
```

E `logout.json` será:

```json
{
    "email": "my@email.com"
}
```

Agora com o teste pronto podemos passar para entender o controller de `logout`. Em `logout` vamos receber o email como parâmetro e um token válido, conforme o teste. Com o email vamos buscar a entidade a ser atualizada e no token vamos verificar a validade do token e se pertence ao usuário correto. Uma vez que as validações estiverem corretas, inativamos seu token com `is_active: false`. Não é tão crítico garantir o `logout` por não se tratar de um código em produção e por ser pouco sensível invalidar um token, caso você queira levar este código a produção, garanta a melhor estratégia com sua equipe de segurança. No nosso controller a primeira coisa que precisamos fazer é verificar se o conteúdo de `Logout` é um email de verdade, para evitar superficialmente `SQL Injection`. Fazemos isso com:

```rust
pub async fn logout(state: web::Data<Clients>, info: web::Json<Logout>) -> impl Responder {
    use regex::Regex;

    let logout_user = info.clone();
    let email_regex = Regex::new("\\w{1,}@\\w{2,}.[a-z]{2,3}(.[a-z]{2,3})?$").unwrap();
    
    if !email_regex.is_match(&logout_user.email) {
        return HttpResponse::BadRequest().finish();
    }
    // ...
}
```

Perceba que caso o campo email não coincida com a regex, nos retornamos `BadRequest`. Além disso, ainda falta implementarmos a struct `Logout` que nos permitirá trocar mensagens com o actor de `DbExecutor`:

```rust
// src/todo_web_api/model/auth.rs
// ...
#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
pub struct Logout {
    pub email: String,
}

impl Message for Logout {
    type Result = Result<User, DbError>;
}

impl Handler<Logout> for DbExecutor {
    type Result = Result<User, DbError>;

    fn handle(&mut self, msg: Logout, _: &mut Self::Context) -> Self::Result {
        use crate::todo_api::db::auth::scan_user;

        scan_user(msg.email, &self.0.get().expect("Failed to open connection"))
    }
}
```

Vale salientar que a função `scan_user` já possui implementação para a `feature` `db-test`. Além disso, é importante ressaltar que a struct `Logout` faz exatamente a mesma coisa que a struct `Login`, exceto pelo fato de que `Login` possui o campo `password`, por isso podemos simplificar a nosso `model` contendo apenas um tipo de `Login/Logout` com o campo `password` opcional. Assim `Login` pode se transformar em `Auth`:

```rust
#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
pub struct Auth {
    pub email: String,
    pub password: Option<String>,
}

impl Message for Auth {
    type Result = Result<User, DbError>;
}

impl Handler<Auth> for DbExecutor {
    type Result = Result<User, DbError>;

    fn handle(&mut self, msg: Auth, _: &mut Self::Context) -> Self::Result {
        use crate::todo_api::db::auth::scan_user;

        scan_user(msg.email, &self.0.get().expect("Failed to open connection"))
    }
}
```

Assim, precisamos atualizar nosso `controller/auth` para utilizar `Auth` em vez de `Login` e `Logout`:

```rust
pub async fn login(state: web::Data<Clients>, info: web::Json<Auth>) -> impl Responder {
    let login_user = info.clone();
    if !is_email_pswd_valids(&login_user.email, &login_user.password.clone().unwrap()) {
        return HttpResponse::BadRequest().finish();
    }

    let resp = state.postgres
        .send(login_user)
        .await;

    match resp {
        Err(e)  => {
            error!("{:?}",e);
            HttpResponse::NoContent().finish()
        },
        Ok(user) => {
            let usr = user.unwrap();
            match usr.verify(info.clone().password.unwrap()) {
                Ok(true) => generate_jwt(usr, state).await,
                Ok(false) => HttpResponse::NoContent().finish(),
                Err(_) => HttpResponse::NoContent().finish()
            }
        }
    }
}

pub async fn logout(state: web::Data<Clients>, info: web::Json<Auth>) -> impl Responder {
    // ...
}
```

Para continuarmos com `logout` precisamos receber o conteúdo do header em um request, fazemos isso adicionando o request aos argumentos de `logout`:

```rust
pub async fn logout(req: HttpRequest, state: web::Data<Clients>, info: web::Json<Auth>) -> impl Responder {...}
```

Para acessarmos o conteúdo que enviamos agora vamos utilizar de uma função chamada `headers`, que retorna um mapa com todos os headers disponíveis. Nosso header de autorizaçnao terá uma cara um pouco diferente, pois se chamará `x-auth` e para obtermos ele basta chamarmos a função `get` que nos retornará um `Option` de `HeaderValue`:

```rust
pub async fn logout(req: HttpRequest, state: web::Data<Clients>, info: web::Json<Auth>) -> impl Responder {
    use regex::Regex;

    let jwt = req.headers().get("x-auth");
    // ...
}
```

Agora vamos fazer uma pequena mudança para tornar mais claro e organizado o controller. A função `is_email_pswd_valids` não pertence a este domínio, assim moveremos ela, e seus testes, para o módulo de core em `src/todo_api/core/mod.rs`, lembre-se de utilizar o `use crate::todo_api::core` no controller.

Em `logout` paramos no match do `email`, mas agora com a informação de email queremos receber informações de `User` para podermos fazer validações para o logout. Fazemos isso utilizando `let resp = state.postgres.send(logout_user.clone())` que se comporta de forma identica ao caso de `login`, e como não temos necessidade desta informação agora, podemos não utilizar o `await` imediatamente. O próximo passo é entender o estado associado ao valor `jwt`, fazemos isso em um `match`, na qual a cláusula `None` é uma resposta de `BadRequest` e a resposta `Some` vai agir sobre `jwt`:

```rust
pub async fn logout(req: HttpRequest, state: web::Data<Clients>, info: web::Json<Auth>) -> impl Responder {
    // ...
    let resp = state.postgres
        .send(logout_user.clone());

    match jwt {
        None => return HttpResponse::BadRequest().finish(),
        Some(jwt) => {
            let jwt_value : JwtValue = serde_json::from_value(decode_jwt(jwt.to_str().unwrap())).expect("failed to parse JWT Value");
            match validate_jwt_date(jwt_value.expires_at) {
                false => 
                    HttpResponse::Unauthorized().finish(),
                true => {
                    validate_jwt_info(jwt_value.email, logout_user.email, resp.await.expect("Failed to read contact info"))
                }
            }
        }
    }
}
```

A primeira coisa que devemos fazer em `Some` é decodificar o `jwt` com a função `decode_jwt`, que recebe como argumento um tipo `&str` (`jwt.to_str().unwrap(`). Nosso uso de `decode_jwt` é bem simples, pois queremos apenas saber se o token ainda é válido, fazemos isso da seguinte forma:

```rust
// src/todo_api/core/mod.rs
pub fn decode_jwt(jwt: &str) -> Value {
    use jsonwebtokens::raw::{TokenSlices, split_token, decode_json_token_slice};

    let TokenSlices {claims, .. } = split_token(jwt).unwrap();
    let claims = decode_json_token_slice(claims).expect("Failed to decode token");
    claims
}
```

A função `decode_jwt` consiste em separar os tokents do argumento `jwt` em partes como `claims` e `headers` e depois aplicar a função `decode_json_token_slice` para extrair o tipo `serde_json::value::Value` de `claims` e retornar `Value`. Essa implementação falharia nosso teste, assim precisamos retornar algum valor aleatório de `Value`, fazemos isso utilizando `serde_json::from_str`:

```rust
#[cfg(feature = "db-test")]
pub fn decode_jwt(jwt: &str) -> Value {
    serde_json::from_str("{\"expires_at\": \"2020-11-02T00:00:00\", \"id\": \"bc45a88e-8bb9-4308-a206-6cc6eec9e6a1\", \"email\": \"my@email.com\"}").unwrap()
}
```

Essa função não necessita grandes testes, já que ela não vai ser alterada com o tempo, mas é sempre bom testar que os valores batem. Assim, o módulo a seguire testa um token Jwt criado no site jwt.io e os valores de seu `claim` sendo transformado em Json pela macro `json!`. Depois disso, testamos a igualdade das partes. Com o teste a seguir vamos quebrar nossa pipeline de testes, pois este teste não utilzia a feature `dbtest` e é executado junto com todos os oturos testes. A solução mais simples para isso é separar testes unitários de testes de integração. Assim, criaremos um target `unit` no Makefile que executará `cargo test --lib`, e os testes de integração serão executados com `cargo test --test lib --features "dbtest"` que executará toda `lib` de `tests/lib`. Note os argumentos `--locked`, `--no-fail-fast` e `-- --test-threads 3`, que representam validar o `Cargo.lock`, não terminar o processo quando algum testes falha e executar os testes em 3 threads, repectivamente. Além disso, os testes `all_args_are_equal_is_accepted` e `all_args_are_not_equal_is_unauth` deverão ser movidos para a pasta de testes de integração `tests`, pois necessitam da feature `db-test`, coloquei eles em um módulo `todo_api_web/validation`.

```rust
#[cfg(test)]
mod decode_jwt {
    use super::decode_jwt;
    use serde_json::json;

    #[test]
    fn decodes_random_jwt() {
        let jwt = decode_jwt("eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6InRlc3QiLCJpYXQiOjE1MTYyMzkwMjJ9.tRF6jrkFnCfv6ksyU-JwVq0xsW3SR3y5cNueSTdHdAg");
        let expected = json!({"sub": "1234567890", "name": "test", "iat": 1516239022 });

        assert_eq!(jwt, expected);
    }
}
```

```sh
int: db
	sleep 2
	diesel setup
	diesel migration run
	cargo test --test lib --no-fail-fast --features "dbtest" -- --test-threads 3
	diesel migration redo


unit:
	cargo test --locked --no-fail-fast --lib -- --test-threads 3

test: unit int
```

```rust
// todo_api_web/validation.rs
use todo_server::todo_api::core::{validate_jwt_info};
use todo_server::todo_api::model::auth::User;
use todo_server::todo_api_web::model::http::Clients;
use actix_web::http::StatusCode;

#[actix_rt::test]
async fn all_args_are_equal_is_accepted() {
    use dotenv::dotenv;
    dotenv().ok();

    let exec = Clients::new();
    let state = actix_web::web::Data::new(exec);

    let user = User::from("my@email.com".to_string(), "pass".to_string());
    let email = "my@email.com".to_string();

    let resp = validate_jwt_info(email.clone(), email, Ok(user), state).await;
    assert_eq!(resp.status(), StatusCode::ACCEPTED);
}

#[actix_rt::test]
async fn all_args_are_not_equal_is_unauth() {
    use dotenv::dotenv;
    dotenv().ok();

    let exec = Clients::new();
    let state = actix_web::web::Data::new(exec);

    let user = User::from("not_my@email.com".to_string(), "pass".to_string());
    let email = "my@email.com".to_string();

    let resp = validate_jwt_info(email.clone(), email, Ok(user), state).await;
    assert_eq!(resp.status(), StatusCode::UNAUTHORIZED);
}
```

Depois de aplicarmos `decode_jwt` ao valor `jwt` transformamos este dado em algo manipulável coma struct que representa seu formato `JwtValue`, `let jwt_value :JwtValue = serde_json::from_value(decode_jwt(jwt.to_str().unwrap()))`.  Com `jwt_value` em mãos podemos checar se a data está correta coma função `validate_jwt_date`, que verificar se a data do momento é inferior ou igual a `expires_at`:

```rust
pub fn validate_jwt_date(jwt_expires: chrono::NaiveDateTime) -> bool {
    chrono::Utc::now().naive_utc() <= jwt_expires
}
```

Com este teste implementado a função `#[cfg(feature = "db-test")] decode_jwt` deve começar a falhar a partir do dia 2 de Novembro, assim, precisamos modificar ela para algo mais próximo de infinito, como mil anos deste momento:

```rust
#[cfg(feature = "db-test")]
pub fn decode_jwt(jwt: &str) -> Value {
    serde_json::from_str("{\"expires_at\": \"3020-11-02T00:00:00\", \"id\": \"bc45a88e-8bb9-4308-a206-6cc6eec9e6a1\", \"email\": \"my@email.com\"}").unwrap()
}
```

Voltando a `validate_jwt_date`, seu tipo de retorno é um booleano, que podemos fazer `match` para validar as respostas. Caso a resposta seja false, respondemos com `HttpResponse::Unauthorized().finish()` e caso seja verdadeiro chamamos um outra função que validará as informações internas, `validate_jwt_info(jwt_value.email, logout_user.email, resp.await.expect("Failed to read contact info"))`. Essa validação consiste em validar a coerência entre todos os `email`s, do Jwt, do Json enviado pelo `DELETE` e o salvo no banco. Note que o `email` salvo no banco é chamado através da concretização da future `resp` com `await`:

```rust
pub fn validate_jwt_info(jwt_email: String, req_email: String, user: Result<User, DbError>) -> HttpResponse {
    match user {
        Err(_) => HttpResponse::Unauthorized().finish(),
        Ok(u) => {
            if u.email == jwt_email && jwt_email == req_email {
                HttpResponse::Accepted().finish()
            } else {
                HttpResponse::Unauthorized().finish()
            }
        }
    }
}
```

A primeira coisa que fazemos em `validate_jwt_info` é um `match` de seu `Result`. Se ocorrer algum erro, o mais fácil é simplesmente responder que a pessoa não tem autorização. Caso não ocorram erros, velrificamos a igualdade entre os emails com `if u.email == jwt_email && jwt_email == req_email `, retornando `HttpResponse::Accepted().finish()` em caso de sucesso e `HttpResponse::Unauthorized().finish()` em caso de falha. Outro ponto importante aqui é que `is_active` deve se tornar falso. E para isso precisamos criar uma nova struct `Inactivate` que comunicará com `DbExecutor` para inativar o email associado:

```rust
#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
pub struct Inactivate {
    pub email: String,
    pub is_active: bool
}
```

Que pode ter uma função `new` que recebe o `email` é já cria a struct com `is_active = false`:

```rust
impl Inactivate {
    pub fn new(email: String) -> Self {
        Self {
            email: email,
            is_active: false
        }
    }
}
```

Depois disso precisamos implementar as traits `Message` e `Handle`:

```rust
impl Message for Inactivate {
    type Result = Result<(), DbError>;
}

impl Handler<Inactivate> for DbExecutor {
    type Result = Result<(), DbError>;

    fn handle(&mut self, msg: Inactivate, _: &mut Self::Context) -> Self::Result {
        use crate::todo_api::db::auth::inactivate_user;

        inactivate_user(msg, &self.0.get().expect("Failed to open connection"))
    }
}
```

Quando a `inactivate_user`, seu corpo é muito parecido com `update_user_jwt_date`, pois encontramos o `target` da mesma forma, mas fazemos update apenas no campo `is_active`:

```rust
pub fn inactivate_user(msg: Inactivate, conn: &PgConnection) -> Result<(), DbError> { 
    use crate::schema::auth_user::dsl::*;

    let target = auth_user.filter(email.eq(msg.email));
    match diesel::update(target)
        .set(is_active.eq(msg.is_active))
        .execute(conn) {
        Ok(_) => Ok(()),
        Err(_) => Err(DbError::TryAgain)
    }
}
```

Além disso, a função de teste é exatamente igual a `update_user_jwt_date`, pois retorna somente um `Ok(())`:

```rust
#[cfg(not(feature = "dbtest"))]
pub fn inactivate_user(msg: Inactivate, conn: &PgConnection) -> Result<(), DbError> { 
    use crate::schema::auth_user::dsl::*;

    let target = auth_user.filter(email.eq(msg.email));
    match diesel::update(target)
        .set(is_active.eq(msg.is_active))
        .execute(conn) {
        Ok(_) => Ok(()),
        Err(_) => Err(DbError::TryAgain)
    }
}

#[cfg(feature = "dbtest")]
pub fn inactivate_user(_msg: Inactivate, _conn: &PgConnection) -> Result<(), DbError> { 
    Ok(())
}
```

Para finalizar a atualização devemos enviar a struct como mensagem com `state.postgres.send`:

```rust
pub async fn validate_jwt_info(jwt_email: String, req_email: String, user: Result<User, DbError>, state: web::Data<Clients>) -> HttpResponse {
    match user {
        Err(_) => HttpResponse::Unauthorized().finish(),
        Ok(u) => {
            if u.email == jwt_email && jwt_email == req_email {
                let inactivate = Inactivate::new(req_email);
                let is_inactive = state.postgres.send(inactivate).await;

                match is_inactive {
                    Ok(_) => HttpResponse::Accepted().finish(),
                    Err(_) => HttpResponse::Unauthorized().finish()
                }
            } else {
                HttpResponse::Unauthorized().finish()
            }
        }
    }
}
```

Algumas coisas mudaram. Agora precisamos passar `state` como argumento `state: web::Data<Clients>` e ao utilizarmos `state.postgres.send(inactivate)`, precisamos de um `await`, que exige que nossa função passe a ser `async` com `pub async fn validate_jwt_info`. Além disso, chamamos a função `new` da struct `Inactivate` com algum dos emails que temos e depois enviamos ela para `DbExecutor` com `let is_inactive = state.postgres.send(inactivate).await;`. Um pattern matching simples em `is_inactive` nos permite responder `Accepted` para o único caso que ocorreu tudo bem. Lembre de incorporar `crate::todo_api::core::model::Inactivate` em seu escopo e de modificar o controller de `logout` para enviar o `state` e utilizar `await` em `validate_jwt_info`:

```rust
pub async fn logout(req: HttpRequest, state: web::Data<Clients>, info: web::Json<Auth>) -> impl Responder {
    // ...

    match jwt {
        None => return HttpResponse::BadRequest().finish(),
        Some(jwt) => {
            let jwt_value : JwtValue = serde_json::from_value(decode_jwt(jwt.to_str().unwrap())).expect("failed to parse JWT Value");
            match validate_jwt_date(jwt_value.expires_at) {
                false => 
                    HttpResponse::Unauthorized().finish(),
                true => {
                    validate_jwt_info(jwt_value.email, logout_user.email, resp.await.expect("Failed to read contact info"), state).await
                }
            }
        }
    }
}
```

Agora, os testes `all_args_are_equal_is_accepted` e `all_args_are_not_equal_is_unauth` em `core/mod.rs` passam a falhar por não receberem o state correto. Como a função `validate_jwt_info` passou a ser `async` sua testabilidade diminuiu, junto com isso vamos utilizar `CLients::new` que depende de `dotenv` estar executando. Para isso, devemos criar um `web::Data<CLients>` que será passado como argumento e disponibilizar um runtime para `async` com `#[actix_rt::test]`:

```rust
#[actix_rt::test]
    async fn all_args_are_equal_is_accepted() {
        use dotenv::dotenv;
        dotenv().ok();

        let exec = Clients::new();
        let state = actix_web::web::Data::new(exec);

        let user = User::from("my@email.com".to_string(), "pass".to_string());
        let email = "my@email.com".to_string();
        
        let resp = validate_jwt_info(email.clone(), email, Ok(user), state).await;
        assert_eq!(resp.status(), StatusCode::ACCEPTED);
    }

    #[actix_rt::test]
    async fn all_args_are_not_equal_is_unauth() {
        use dotenv::dotenv;
        dotenv().ok();

        let exec = Clients::new();
        let state = actix_web::web::Data::new(exec);

        let user = User::from("not_my@email.com".to_string(), "pass".to_string());
        let email = "my@email.com".to_string();

        let resp = validate_jwt_info(email.clone(), email, Ok(user), state).await;
        assert_eq!(resp.status(), StatusCode::UNAUTHORIZED);
    }
```

Por último, precisamos dar uma organizada no nosso código.

## Refatorando

Existem três coisas que eu gostaria de refatorar no momento. A primeira é o módulo de erros `db/error.rs`, que está totalmente deslocado. A segunda é mover o `core/model.rs` para `model/core.rs`, pois creio que agora já cresceu bastante. E a terceira é encontrar um nome melhor para `UpdateDate`,  como `UpdateUserStatus`. Começando pela terceira, selecionei para que meu editor de texto encontrasse todos os casos de `UpdateDate` e substituisse eles por `UpdateUserStatus` sem grandes conflitos. Depois disso, vamos mover o módulo de erros. Para iniciarmos o processo, precisamos mover a definição do módulo de `db/mod.rs` para `model/mod.rs`:

```rust
//db/mod.rs
pub mod helpers;
pub mod todo;
pub mod auth;

// model/mod.rs
use rusoto_dynamodb::AttributeValue;
use std::collections::HashMap;
use uuid::Uuid;

pub mod auth;
pub mod error;
// ...
```

Movemos todo o arquivo e precisamores modificar o caminho do `use` deste arquivo nos seguintes arquivos:

* `src/todo_api/core` em `mod.rs` e `model.rs`.
* `src/todo_api/db/auth.rs`
* `src/todo_api_web/model/auth.rs`

São mudanças bastante simples, basta substituir o `db` pelo `model` nos caminhos dos `use`. E para a segunda mudança, vamos criar o módulo `core` em `model/mod.rs`  com `pub mod core` e mover o arquivo `core/model.rs` para `model/core.rs`. Vamos modificar os mesmos arquivos que modificamos em `db/error`, a única diferença é que a função `generate_jwt` incorporava `Jwt` em seu escopo de forma individual. Executando nossos testes com `make test` está tudo ok e podemos continuar para implementar o requerimento de jwt nas chamadas dos endpoints que já temos.

[Anterior](04-serving.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](06-middleware.md)