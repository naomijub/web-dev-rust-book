[Anterior](03-get.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](05-auth.md)

# Tornando nosso serviço mais realístico

Agora vamos aplicar uma série de mudanças em nosso servidor para deixá-lo mais robusto. Algumas dessas mudanças incluem sistemas de logs, conteinerizar a aplicação, tornar ela fault tolerante, headers padrões e mais. Para isso, vamos começar com o mais simples e indispensável, o sistema de logs.

## Aplicando logs

O primeiro passo para começarmos a entender logs em Rust é darmos uma olhada na crate responsável por isso. A crate que vamos utilizar é a `log = "0.4.8"`, que implementa sua lógica de logs de acordo com a ideia de que um log consiste em um `alvo`, um `nível` e um `corpo`. O alvo é uma string que define o caminho do módulo no qual o requerimento do log é necessário. O nível é a severidade do log, `error`, `warn`, `info`, `debug` e `trace`, e o corpo é o conteúdo que o log apresenta. 

A crate que vamos utilizar nos disponibiliza cinco macros para isso: ` error!, warn!, info!, debug!, trace!`, dentre as quais `error` é a mais severa e `trace` a menos severa. As macros funcionam de forma muito similar ao `println!`, assim a forma de utilizá-las é bastante intuitiva. Outra questão importante é que o sistema de logs deve ser inicializado apenas uma vez por outra crate, a mais comum delas é a `env_logger = "0.9.0"`. Um exemplo rápido de como ficaria a combinação dessas duas é:

```rust
#[macro_use]
extern crate log;

fn main() {
    env_logger::init();

    info!("starting up");

    // ...
}
```

### Inicializando o sistema de Logs

Para inicializar nosso sistema de logs, precisamos adicionar a crate `env_logger` ao nosso `[dependencies]` do `Cargo.toml`, o `env_logger = "0.9.0"`. Com a crate disponível, podemos importar o `env_logger` para o contexto do arquivo `main.rs` com `use env_logger;` e inicializá-lo com `env_logger::init()` conforme o código a seguir:

```rust
// ...
use env_logger;

#[actix_rt::main]
async fn main() -> std::io::Result<()> {
    env_logger::init();
    // ...
}
```

Com isso o código parece compilar, mas não conseguimos ver logs no console quando executamos um `curl`. Isso se deve ao fato de que precisamos informar ao `actix_web` que queremos que logs de algum nível sejam disponibilizados. Para isso, devemos incluir a linha `std::env::set_var("RUST_LOG", "actix_web=info");` antes de `env_logger::init();` na função `main` para habilitar logs de `error` a `info`. Além disso, precisamos disponibilizar o middleware `Logger` com a forma como queremos o log, note que o middleware pertence à crate `actix_web`em `use actix_web::middleware::Logger;`:

```rust
// ...
use actix_web::middleware::Logger;
use env_logger;
// ...

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    std::env::set_var("RUST_LOG", "actix_web=info");
    env_logger::init();
    create_table().await;
    
    HttpServer::new(|| {
        App::new()
            .wrap(Logger::new("IP:%a DATETIME:%t REQUEST:\"%r\" STATUS: %s DURATION:%D"))
            .configure(app_routes)
            .default_service(web::to(|| HttpResponse::NotFound()))
    })
    .workers(num_cpus::get() - 2)
    .bind(("localhost", 4004))
    .unwrap()
    .run()
    .await
}
```

Se fizermos um `POST curl` agora no endpoint `/api/create` vamos ver o seguinte log no terminal aonde o servidor está rodando:

```
[2020-02-08T01:41:32Z INFO  actix_web::middleware::logger] IP:127.0.0.1:54089 DATETIME:2020-02-07T22:41:32-03:00 REQUEST:"POST /api/create HTTP/1.1" STATUS: 201 DURATION:33.976000
```

Note que o formato após os colchetes `[...]` é igual ao que definimos no middleware de `Logger::new("IP:%a DATETIME:%t REQUEST:\"%r\" STATUS: %s DURATION:%D")`, assim podemos entender alguns dos parâmetros que estamos passando:

* `%a` é o IP do request.
* `%t` é o DateTime do request.
* `%r` é o método (`POST` no caso) seguido do endpoint (`/api/create`) e o protocolo usado.
* `%s` é o status de retorno do request.
* `%D` é a duração total do request, em milisegundos.

Algumas outras variáveis disponíveis nesse middleware são:

* `%t` horário no qual o request começou a ser processado.
* `%P` o ID do processo filho que serviu o request.
* `%b` tamanho da resposta em bytes (inclui os headers).
* `%T` duração do request em segundos com fração float de `.06f`.
* `%{FOO}i` headers[‘FOO’] do request.
* `%{FOO}o` headers[‘FOO’] da response.
* `%{FOO}e` valor da variável de ambiente `FOO`, `os.environ["FOO"]`.
Algumas outras variávels disponíveis neste middleware são:

### Adicionando logs

Para adicionar os logs ao nosso código, vamos utilizar duas macros `error!` e `debug!`. Para isso, precisamos adicionar `log = "0.4"` ao nosso `[dependencies]` no `Cargo.toml`. A função de `debug` deverá nos apoiar com resultados no ambiente de desenvolvimento, enquanto a função de `error!` será exibir os erros no console. Para isso, usaremos o código `use log::{error, debug};`. Um bom local para inicializar é no `create_table`, a primeira função que nosso código executa. Para modo debug, utilize a env `std::env::set_var("RUST_LOG", "debug");`:

```rust
use log::{debug, error};
// ...
pub async fn create_table() {
    let client = get_client().await;
    match client.list_tables().send().await {
        Ok(list) => {
            match list.table_names {
                Some(table_vec) => {
                    if table_vec.len() > 0 {
                         error!("Table already exists and has more then one item");
                    } else {
                        create_table_input(&client).await
                    }
                }
                None => create_table_input(&client).await,
            };
        }
        Err(_) => {
            create_table_input(&client).await;
        }
    }
}

async fn create_table_input(client: &Client) {
    let table_name = TODO_CARD_TABLE.to_string();
    let ad = build_attribute_definition();
    let ks = build_key_schema();
    let pt = build_provisioned_throughput();

    match client
        .create_table()
        .table_name(table_name)
        .key_schema(ks)
        .attribute_definitions(ad)
        .provisioned_throughput(pt)
        .send()
        .await
    {
        Ok(output) => {
            debug!("Table created {:?}", output);
        }
        Err(error) => {
            error!("Could not create table due to error: {:?}", error);
        }
    }
}
```

Outro lugar em que podemos aplicar logs é no arquivo `src/todo_api/db/todo.rs`, pois as funções de `put` e `get` são bastante suscetíveis a erros. Assim podemos modificar o arquivo para:

```rust
// ...
use log::{debug, error};

#[cfg(not(feature = "dynamo"))]
pub async fn put_todo(client: &Client, todo_card: TodoCardDb) -> Option<uuid::Uuid> {
    match client
        .put_item()
        .table_name(TODO_CARD_TABLE.to_string())
        .set_item(Some(todo_card.clone().into()))
        .send()
        .await
    {
        Ok(_) => {
            debug!("item created with id {:?}", todo_card.id);
            Some(todo_card.id)
        }
        Err(e) => {
            error!("error when creating item {:?}", e);
            None
        }
    }
}


#[cfg(not(feature = "dynamo"))]
pub async fn get_todos(client: &Client) -> Option<Vec<TodoCard>> {
    // ...
    match scan_output {
        Ok(dbitems) => {
            let res = adapter::scanoutput_to_todocards(dbitems)?.to_vec();
            debug!("Scanned {:?} todo cards", dbitems);
            Some(res)
        }
        Err(e) => {
            error!("Could not scan todocards due to error {:?}", e);
            None
        }
    }
}
```

Note que nos casos de `Err` agora estamos logando o motivo com `e`. O último passo para este momento é adicionar logs aos controllers em `src/todo_api_web/controllers/todo.rs`:

```rust
use log::{error};
// ...

#[post("/api/create")]
pub async fn create_todo(info: web::Json<TodoCard>) -> impl Responder {
    let id = Uuid::new_v4();
    let todo_card = adapter::todo_json_to_db(info, id);
    let client = get_client().await;
    match put_todo(&client, todo_card).await {
        None => {
            error!("Failed to create todo card");
            HttpResponse::BadRequest().body(ERROR_CREATE)
        }
        Some(id) => HttpResponse::Created()
            .content_type(ContentType::json())
            .body(serde_json::to_string(&TodoIdResponse::new(id)).expect(ERROR_SERIALIZE)),
    }
}

#[get("/api/index")]
pub async fn show_all_todo() -> impl Responder {
    let client = get_client().await;
    let resp = get_todos(&client).await;
    match resp {
        None => {
            error!("Failed to read todo cards");
            HttpResponse::InternalServerError().body(ERROR_READ)
        }
        Some(cards) => HttpResponse::Ok()
            .content_type(ContentType::json())
            .body(serde_json::to_string(&TodoCardsResponse { cards }).expect(ERROR_SERIALIZE)),
    }
}
```

Note que adicionamos somente a opção de `error` já que o `None => {...}` é a única resposta que pode conter diversas razões, pelo fato do `Some` já estar mapeado em `put_todo` e `get_todos`.

## Incluindo Docker

Como o foco deste livro não é docker e ele não é um requisito para entender o livro, vou mostrar o código e explicar um pouco o que está acontecendo. Assim vamos começar por um `Dockerfile` extremamente simples.

```Dockerfile
FROM rustlang/rust:nightly

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

COPY . /usr/src/app

CMD ["cargo", "build", "-q"]
```

A primeira diretiva, a `FROM`, tem como objetivo definir a imagem base para nosso contêiner. Nesse caso, estamos utilizando uma versão `nightly` do Rust, pois a versão stable não era compatível com a versão da minha máquina quando escrevi este livro. Depois disso, temos a diretiva `RUN`, que executa algum comando, no nosso caso a criação da pasta `/usr/src/app`, e já definimos essa pasta como o diretório que vamos utilizar com `WORKDIR`. Depois disso, copiamos todo nosso código para nosso diretório com `COPY` e executamos um comando do cargo, o `build`, para construir nossa aplicação, `cargo build -q` com `CMD`. Outra opção de `Dockerfile` com otimização para builds repetidos é:

```Dockerfile
FROM rust:latest

RUN mkdir -p /usr/src/
WORKDIR /usr/src/
RUN USER=root cargo new --bin app
WORKDIR /app

COPY ./Cargo.lock ./Cargo.lock
COPY ./Cargo.toml ./Cargo.toml
COPY ./tests ./tests

RUN cargo build --release
RUN rm src/*.rs

COPY ./src ./src

CMD ["cargo", "build", "--release"]
```

O objetivo desse segundo `Dockerfile` é diminuir o tempo de execução do contêiner ao cachear as dependências do app e somente atualizar o cache a partir do `COPY ./src ./src`.

Com este container pronto, podemos começar a pensar em como utilizar os dois containers (DynamoDB e `todo_server`) em conjunto. Faremos isso com `docker-compose.yml`:

```yml
version: '3.8'
services:
  dynamodb-local:
    command: "-jar DynamoDBLocal.jar -sharedDb -dbPath ./data"
    image: amazon/dynamodb-local
    container_name: dynamodb-local
    ports:
      - "8000:8000"
    volumes:
      - "./docker/dynamodb:/home/dynamodblocal/data"
    working_dir: /home/dynamodblocal
  web:
    build:
      context: .
      dockerfile: Dockerfile
    command: cargo run
    ports:
      - "4000:4000"
    depends_on:
      - "dynamodb-local"
    links:
      - "dynamodb-local"
    environment:
      # Since we are using dynamodb local, the IAM authentication mechanism is not used at all. 
      # That is, whichever credentials you provide, it will be accepted
      AWS_ACCESS_KEY_ID: 'MYID'
      AWS_SECRET_ACCESS_KEY: 'MYSECRET'
      AWS_REGION: 'us-east-1'
      DYNAMODB_ENDPOINT: 'dynamodb-local'
```
Nosso `docker-compose` precisa de duas chaves principais: `version`, que corresponde à versão do compose, `services`, que corresponde aos contêineres que vamos rodar. Em `services`, precisamos declarar dois contêineres `web`, os quais conterão nossa aplicação e o contêiner `dynamodb`, que conterá a imagem do **DynamoDB** e veio (desse tutorial)[https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.DownloadingAndRunning.html]. O contêiner `dynamodb` possui as seguintes chaves:

* `container_name`: é o nome do contêiner, no nosso caso `dynamodb`.
* `image`: a fonte da imagem que estamos utilizando, no caso do DynamoDB é `amazon/dynamodb-local`.
* `ports`: o mapeamento de portas de dentro do contêiner para fora, `8000:8000`.
* `volumes`: volumes disponíveis para o dynamo utilizar, `dynamodata:/home/dynamodblocal/`.
* `working_dir`: diretório no qual o dynamo executará, `/home/dynamodblocal/`.
* `command`: para inicializar o dynamo `"-jar DynamoDBLocal.jar -sharedDb -dbPath ."`.

Depois disso temos o `web` que irá rodar a todo API, que não vou repetir algumas chaves:

* `build`: o contexto de criação da imagem, `context: .`. No caso, estamos passando um dockerfile chamado `Dockerfile` `dockerfile: Dockerfile`.
* `command`: executamos o comando `cargo run` para essa aplicação.
* `environment`: para executar o DynamoDB dessa forma precisamos adicionar algumas variáveis de ambiente para que o `client` configure suas credenciais,
de acordo com https://docs.aws.amazon.com/sdk-for-rust/latest/dg/dynamodb-local.html.
    - `AWS_ACCESS_KEY_ID=AKIDLOCALSTACK`
    - `AWS_SECRET_ACCESS_KEY=localstacksecret`
    - `AWS_REGION=us-east-1`
    - `DYNAMODB_ENDPOINT=dynamodb-local` 
Usaremos a variável de ambiente `DYNAMODB_ENDPOINT` para saber qual address iremos usar quando inicializarmos
o dynamodb client na nossa API. Faremos a seguinte mudança na função `get_client`:

```rust
// src/todo_api/db/helpers.rs
pub async fn get_client() -> Client {
    let config = aws_config::load_from_env().await;

    let addr = if let Ok(db_endpoint) = std::env::var("DYNAMODB_ENDPOINT") {
        format!("http://{}:8000", db_endpoint)
    } else {
        "http://0.0.0.0:8000".to_string()
    };

    let dynamodb_local_config = aws_sdk_dynamodb::config::Builder::from(&config)
        .endpoint_resolver(Endpoint::immutable(addr.parse().expect("Invalid URI")))
        .build();
    Client::from_conf(dynamodb_local_config)
}
```
* `depends_on`: define a ordem na qual os serviços devem ser inicializados, assim `dynamodb` é inicializado antes de `web`
* `links`: forma legada de fazer com que dois serviços estejam conectados, atualmente bastaria o `networks`, mas coloquei como exemplo. No caso de `links` e `networks` estarem definidos, é preciso que ambos estejam na mesma rede.

Se tivéssemos as configurações de produção, poderíamos criar a feature `compose` para utilizar com o `docker-compose`. Se executarmos o código agora com `docker-compose up --build` e, em seguida, um `curl`, tudo voltará a funcionar como antes. Outra coisa que podemos fazer agora é atualizar nosso Makefile para incluir o `docker-compose`:

```sh
db:
	docker run -p 8000:8000 amazon/dynamodb-local

test:
	cargo test --features "dynamo"

run-local:
	cargo run --features "dynamo"

run:
	docker-compose up --build

down:
	docker-compose down
```

## Headers padrões

Outro ponto que acredito ser importante é o uso de headers para identificar os requests nos logs. Costumo ver o padrão de um header chamado `x-request-id` cujo valor é um `uuid`. Para implementarmos esse padrão com o actix, precisamos utilizar um middleware que felizmente a equipe do actix já disponibilizou para nós, o `actix_web::middleware::DefaultHeaders`. Para isso, precisamos disponibilizá-lo no escopo com `use` e depois passar essa informação para um `wrap`. A forma de utilizar esses headers padrões é `DefaultHeaders::new().header("X-Version", "0.2")`, isto é, criamos um novo header com `DefaultHeaders::new()` e depois chamamos a função `header` para adicionar um header com os argumentos-chave e valor do tipo string:

```rust
// src/main.rs
// ...
HttpServer::new(|| {
    App::new()
        .wrap(DefaultHeaders::new().add(("x-request-id", Uuid::new_v4().to_string())))
        .wrap(Logger::new(
            "IP:%a DATETIME:%t REQUEST:\"%r\" STATUS: %s DURATION:%D",
        ))
        .configure(app_routes)
    })
// ...
```

Além disso, precisamos definir o header no `Logger`, para isso usamos a chave `X-REQUEST-ID:%{x-request-id}o` após a `DURATION`, pois somente assim o valor de `x-request-id` será logado:

```rust
// ...
HttpServer::new(|| {
    App::new()
        .wrap(DefaultHeaders::new().add(("x-request-id", Uuid::new_v4().to_string())))
        .wrap(Logger::new("IP:%a DATETIME:%t REQUEST:\"%r\" STATUS: %s DURATION:%D X-REQUEST-ID:%{x-request-id}o"))
        .configure(app_routes)
    })
// ...
```

Um exemplo de resposta seria:

```
[2020-02-08T23:10:58Z INFO  actix_web::middleware::logger] IP:172.21.0.1:52686 DATETIME:2020-02-08T23:10:58Z REQUEST:"POST /api/create HTTP/1.1" STATUS: 201 DURATION:166.921700 X-REQUEST-ID=bd15de62-1ba6-4d43-89ca-4f89418
```

## Adicionando o cliente ao estado da API

Nosso próximo passo vem de uma necessidade de refactor e preparação para o código futuro. Esse refactor consiste em retirar a declaração de `let cliente = client();` de todos os códigos envolvendo banco de dados e passá-los como argumentos. Uma das vantagens disso é caso decidamos ter mais clientes de algum tipo de serviço como outros bancos de dados ou S3. Para fazermos isso, vamos criar uma nova struct chamada `Clients` que conterá o campo `dynamo` e depois a passaremos como argumento para o `HttpServer` via função `data`.

Assim, nosso primeiro passo é descrever a o modelo de `Clients` em `src/todo_api_web/model/http.rs`:

```rust
use aws_sdk_dynamodb::Client;

use crate::todo_api::db::helpers::get_client;

#[derive(Clone)]
pub struct Clients {
    pub dynamo: Client,
}

impl Clients {
    pub async fn new() -> Self {
        Self {
            dynamo: get_client().await,
        }
    }
}

```

Agora podemos utilizar a função `app_data` em `HttpServer` para passar Clients como argumento. Fazemos isso com `Clients::new()`:

```rust
// ...
use todo_server::{
    todo_api::db::helpers::create_table,
    todo_api_web::{model::http::Clients, routes::app_routes},
};

#[actix_web::main]
async fn main() -> Result<(), std::io::Error> {
    std::env::set_var("RUST_LOG", "actix_web=info");
    env_logger::init();

    let client = web::Data::new(Clients::new().await);
    create_table(&client.dynamo.clone()).await;

    HttpServer::new(move|| {
        App::new()
            .app_data(client.clone())
            .wrap(DefaultHeaders::new().add(("x-request-id", Uuid::new_v4().to_string())))
            .wrap(Logger::new("IP:%a DATETIME:%t REQUEST:\"%r\" STATUS: %s DURATION:%D X-REQUEST-ID:%{x-request-id}o"))
            .configure(app_routes)
    })
    .workers(num_cpus::get() - 2)
    .max_connections(30000)
    .bind(("0.0.0.0", 4000))
    .unwrap()
    .run()
    .await
}
// ...
```

Com isso temos `Clients` disponível no nos nossos controllers, para isso adicionamos o estado com `state: web::Data<Clients>`:

```rust
// ...
use crate::todo_api_web::model::http::Clients;

#[post("/api/create")]
pub async fn create_todo(state: web::Data<Clients>, info: web::Json<TodoCard>) -> impl Responder {
    let id = Uuid::new_v4();
    let todo_card = adapter::todo_json_to_db(info, id);
    let client = state.dynamo.clone();
//...
}

#[get("/api/index")]
pub async fn show_all_todo(state: web::Data<Clients>) -> impl Responder {
    let client = state.dynamo.clone();
    //...
}
```

As funcões `put_todo` e `get_todos` ja esperam um argumento do tipo `aws_sdk_dynamodb::Client` então
não será preciso modificar elas.

Feito isso, devemos adicionar o novo client a todos os testes de integração, pois esse argumento é esperado nas funções de controller. Um exemplo seria:

```rust
    #[actix_web::test]
    async fn test_todo_cards_count() {
        let client = web::Data::new(Clients::new().await);
        let mut app =
            test::init_service(App::new().app_data(client.clone()).configure(app_routes)).await;
    //...
    }
```

### Serializando o Response

Até o momento estávamos utilizando o formato de criação de `HttpResponse` da seguinte maneira `HttpResponse::Ok().content_type("application/json").body(serde_json::to_string(&struct).expect("Failed to serialize todo cards"))`, mas existe uma forma que pode simplificar nossa vida por nos permitir delegar a chamada de `serde_json`. Esse formato substitui o `.body(...)` por `.json(...)`. A vantagem de se utilizar esse formato é que ele reduz a quantidade de código que nós devemos manter, delegando ao actix essa responsabilidade. Nos capítulos introdutórios do livro, falamos que o actix estava com muita vantagem em relação a outros frameworks nos benchmarks da TechEmpower, porém, no caso de serialização JSON, existem alguns frameworks C/C++ à sua frente, inclusive a crate `hyper`. O Objetivo de `body` é principalmente enviar mensagens sem dados estruturados ou estruturados em outros formatos como [Edn](https://crates.io/crates/edn-rs).

Com esse pequeno refactor, nossos controllers de `todo` serão modificados para o seguinte formato:

```rust
// src/todo_web_api/controller/todo.rs
// ...
#[post("/api/create")]
pub async fn create_todo(state: web::Data<Clients>, info: web::Json<TodoCard>) -> impl Responder {
    let id = Uuid::new_v4();
    let todo_card = adapter::todo_json_to_db(info, id);
    let client = state.dynamo.clone();

    match put_todo(&client, todo_card).await {
        None => {
            error!("Failed to create todo card {}", ERROR_CREATE);
            HttpResponse::BadRequest().body(ERROR_CREATE)
        }
        Some(id) => HttpResponse::Created()
            .content_type(ContentType::json())
            .json(TodoIdResponse::new(id)),
    }
}

#[get("/api/index")]
pub async fn show_all_todo(state: web::Data<Clients>) -> impl Responder {
    let client = state.dynamo.clone();
    let resp = get_todos(&client).await;
    match resp {
        None => {
            error!("Failed to read todo cards");
            HttpResponse::InternalServerError().body(ERROR_READ)
        }
        Some(cards) => HttpResponse::Ok()
            .content_type(ContentType::json())
            .json(TodoCardsResponse { cards }),
    }
}
```

Com isso, nosso código está pronto para receber novos clientes e nós podemos começar a pensar em autenticação.

[Anterior](03-get.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](05-auth.md)