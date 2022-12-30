[Anterior](./03-get.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](./05-auth.md)

# Tornam nosso serviço mais realístico

Agora vamos aplicar uma série de mudanças em nosso servidor para deixá-lo mais robusto. Algumas dessas mudanças incluem sistemas de logs, conteinerizar a aplicação, tornar ela fault tolerante, headers padrões e mais. Para isso, vamos começar com o mais simples e indispensável, o sistema de logs.

## Aplicando logs

O primeiro passo para começarmos a entender logs em Rust é darmos uma olhada na crate responsável por isso. A crate que vamos utilizar é a `log = "0.4.8"`, que implementa sua lógica de logs de acordo com a ideia de que um log consiste em um `alvo`, um `nível` e um `corpo`. O alvo é uma string que define o caminho do módulo no qual o requerimento do log é necessário. O nível é a severidade do log, `error`, `warn`, `info`, `debug` e `trace`, e o corpo é o conteúdo que o log apresenta. 

A crate que vamos utilizar nos disponibiliza cinco macros para isso: ` error!, warn!, info!, debug!, trace!`, dentre as quais `error` é a mais severa e `trace` a menos severa. As macros funcionam de forma muito similar ao `println!`, assim a forma de utilizá-las é bastante intuitiva. Outra questão importante é que o sistema de logs deve ser inicializado apenas uma vez por outra crate, a mais comum delas é a `env_logger = "0.7.1"`. Um exemplo rápido de como ficaria a combinação dessas duas é:

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

Para inicializar nosso sistema de logs, precisamos adicionar a crate `env_logger` ao nosso `[dependencies]` do `Cargo.toml`, o `env_logger = "0.7.1"`. Com a crate disponível, podemos importar o `env_logger` para o contexto do arquivo `main.rs` com `use env_logger;` e inicializá-lo com `env_logger::init()` conforme o código a seguir:

```rust
// ...
use env_logger;

#[actix_rt::main]
async fn main() -> std::io::Result<()> {
    env_logger::init();
    // ...
}
```

Com isso o c;odigo parece compilar, mas não conseguimos ver logs no console quando executamos um `curl`. Isso se deve ao fato de que precisamos informar ao `actix_web` que queremos que logs de algum nível sejam disponibilizados. Para isso, devemos incluir a linha `std::env::set_var("RUST_LOG", "actix_web=info");` antes de `env_logger::init();` na função `main` para habilitar logs de `error` a `info`. Além disso, precisamos disponibilizar o middleware `Logger` com a forma como queremos o log, note que o middleware pertence à crate `actix_web`em `use actix_web::middleware::Logger;`:

```rust
// ...
use actix_web::middleware::Logger;
use env_logger;
// ...

#[actix_rt::main]
async fn main() -> std::io::Result<()> {
    std::env::set_var("RUST_LOG", "actix_web=info");
    env_logger::init();
    create_table();
    
    HttpServer::new(|| {
        App::new()
        .wrap(Logger::new("IP:%a DATETIME:%t REQUEST:\"%r\" STATUS: %s DURATION:%D"))
        .configure(app_routes)
    })
    .workers(num_cpus::get() + 2)
    .bind("127.0.0.1:4000")
    .unwrap()
    .run()
    .await
}

```

Se fizermos um `POST curl` agora no endpoint `/api/create` vamos receber o seguinte log:

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

pub fn create_table() {
    let client = client();
    let list_tables_input: ListTablesInput = Default::default();

    match client.list_tables(list_tables_input).sync() {
        Ok(list) => {
            match list.table_names {
                Some(table_vec) => {
                    if table_vec.len() > 0 {
                        error!("Table already exists and has more then one item");
                    } else {
                        create_table_input()
                    }
                }
                None => create_table_input(),
            };
        }
        Err(_) => {
            create_table_input();
        }
    }
}

fn create_table_input() {
    // ...

    match client.create_table(create_table_input).sync() {
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
pub fn put_todo(todo_card: TodoCardDb) -> Option<Uuid> {
    // ...

    match client.put_item(put_item).sync() {
        Ok(_) => {
            debug!("item created with id {:?}", todo_card.id);
            Some(todo_card.id)
        },
        Err(e) => {
            error!("error when creating item {:?}", e);
            None
        }
    }
}

#[cfg(not(feature = "dynamo"))]
pub fn get_todos() -> Option<Vec<TodoCard>> {
    // ...

    match client.scan(scan_item).sync() {
        Ok(resp) => {
            let todocards = adapter::scanoutput_to_todocards(resp);
            debug!("Scanned {:?} todo cards", todocards);
            Some(todocards)
        },
        Err(e) => {
            error!("Could not scan todocards due to error {:?}", e);
            None
        },
    }
}
```

Note que nos casos de `Err` agora estamos logando o motivo com `e`. O úmtimo passo para este momento é adicionar logs aos controllers em `src/todo_api_web/controllers/todo.rs`:

```rust
use log::{error};
// ...

pub async fn create_todo(info: web::Json<TodoCard>) -> impl Responder {
    let todo_card = adapter::todo_json_to_db(info, uuid::Uuid::new_v4());

    match put_todo(todo_card) {
        None => {
            error!("Failed to create todo card");
            HttpResponse::BadRequest().body("Failed to create todo card")
        },
        Some(id) => HttpResponse::Created()
            .content_type("application/json")
            .body(serde_json::to_string(&TodoIdResponse::new(id)).expect("Failed to serialize todo card"))
    }
}

pub async fn show_all_todo() -> impl Responder {
    match get_todos() {
        None => {
            error!("Failed to read todo cards");
            HttpResponse::InternalServerError().body("Failed to read todo cards")
        },
        Some(todos) => HttpResponse::Ok()
            .content_type("application/json")
            .body(serde_json::to_string(&TodoCardsResponse{cards: todos}).expect("Failed to serialize todo cards")),
    }
}
```

Note que adicionamos somente a opção de `error` já que o `None => {...}` é a única resposta que pode conter diversas razões, pelo fato do `Some` já estar mapeado em `put_todo` e `get_todos`.

## Tornando nosso sistema tolerância a falha

Erlang possui um sistema de tolerância a falhas inspirado em sistemas de atores bastante poderosos e versáteis. Assim a comunidade Rust desenvolveu uma crate que ajuda esse sistema. A crate é chamada de `bastion` e depende de outra crate chamada `fort`, que disponibiliza o runtime, para que `bastion` funcione. **Lembre-se de que adicionar runtimes implica em um aumento de consumo de memória e tamanho de executável**, mas vamos seguir com essa abordagem. Assim adicionamos as duas crates ao nosso `[dependencies]` do `Cargo.toml`:

```toml
[dependencies]
# ...
fort = "0.3"
bastion = "0.3"
```

Para utilizarmos a crate `fort`, precisamos habilitar o runtime de `bastion` com a macro `#[fort::root]` e modificar a chamada da `main`. Fazemos isso modificando a antiga função `main` para se chamar `web_main` ou `actix_main`, e chamamos ela da instância `main` na qual o `bastion` está disponível:

```rust
// ...
use bastion::prelude::*;


#[actix_rt::main]
async fn web_main() -> Result<(), std::io::Error> {
    std::env::set_var("RUST_LOG", "actix_web=info");
    env_logger::init();
    create_table();
    
    HttpServer::new(|| {
        App::new()
        .wrap(Logger::new("IP:%a DATETIME:%t REQUEST:\"%r\" STATUS: %s DURATION:%D"))
        .configure(app_routes)
    })
    .workers(num_cpus::get() + 2)
    .bind("127.0.0.1:4000")
    .unwrap()
    .run()
    .await
}

#[fort::root]
async fn main(_: BastionContext) -> Result<(), ()> {
    let _ = web_main();
    Ok(())
}
```

Note que a função `main` também é declarada como `async`, isso é um requerimento de `fort::root`. Assim, ao executarmos `cargo run` o sistema vai se reiniciar mesmo que utilizemos `crtl+c`. Inclusive outro teste que podemos fazer é adicionar um `panic!` antes do `Ok(())`, que interromperá a thread a cada ciclo:

```rust
#[fort::root]
async fn main(_: BastionContext) -> Result<(), ()> {
    let _ = web_main();
    panic!("Holy shit");
    Ok(())
}
```

Note que neste caso, ao tentarmos utilizar um extra `ctrl+c` recebemos o seguinte erro:

```
Panic in Arbiter thread.
thread 'bastion-async-thread' panicked at 'env_logger::init should not be called after logger initialized: SetLoggerError(())', src/libcore/result.rs:1165:5
```

Com isso, podemos concluir que as funções `std::env::set_var("RUST_LOG", "actix_web=info")`, `env_logger::init()` e `create_table()` não precisam estar dentro da função `web_main()`:

```rust
#[actix_rt::main]
async fn web_main() -> Result<(), std::io::Error> {    
    HttpServer::new(|| {
        App::new()
        .wrap(Logger::new("IP:%a DATETIME:%t REQUEST:\"%r\" STATUS: %s DURATION:%D"))
        .configure(app_routes)
    })
    .workers(num_cpus::get() + 2)
    .bind("127.0.0.1:4000")
    .unwrap()
    .run()
    .await
}

#[fort::root]
async fn main(_: BastionContext) -> Result<(), ()> {
    std::env::set_var("RUST_LOG", "actix_web=info");
    env_logger::init();
    create_table();
    
    let _ = web_main();

    Ok(())
}
```

Na macro `fort::root`, existe um atributo a mais que podemos passar, o `redundancy`. Este atributo espera um valor do tipo inteiro positivo e, para utilizá-lo, basta adicionar `#[fort::root(redundancy = 2)]`. O atributo `redundancy` corresponde ao número de elementos que este grupo vai ter, no nosso caso a quantidade de `web_main()` que vamos iniciar, valor padrão de `redundancy` é `1`.

No momento, precisamos ter cuidado, pois já estabelecemos uma quantidade de `workers` igual a `num_cpus::get() + 2` e isso faz com que não sobrem muitos cores para iniciarmos processos. Uma possível solução para o `redundancy` seria iniciá-lo em diferentes máquinas distribuídas, mas essa crate de `bastion` ainda não está estável. Outra possível forma de limitar o tamanho dos `workers` para poder tirar proveito do `redundancy` é definir no `HttpServer` a quantidade máxima de conexões que cada `worker` pode estabelecer com a função `maxconn`, seu valor padrão é `25k`. Um exemplo fictício seria:

```rust
#[actix_rt::main]
async fn web_main() -> Result<(), std::io::Error> {    
    HttpServer::new(|| {
        App::new()
        .wrap(Logger::new("IP:%a DATETIME:%t REQUEST:\"%r\" STATUS: %s DURATION:%D"))
        .configure(app_routes)
    })
    .workers(num_cpus::get() - 2)
    .maxconn(30000)
    .bind("127.0.0.1:4000")
    .unwrap()
    .run()
    .await
}

#[fort::root(redundancy = 10)]
async fn main(_: BastionContext) -> Result<(), ()> {
    std::env::set_var("RUST_LOG", "actix_web=info");
    env_logger::init();
    create_table();
    
    let _ = web_main();

    Ok(())
}
```
> A crate que, no momento em que escrevi este livro, estava sendo desenvolvida para instâncias remotas do `bastion` pode ser encontrada no link https://github.com/bastion-rs/artillery.

Agora podemos evoluir nosso código para facilitar nossa vida quando executamos um processo que se recusa a terminar, faremos isso com containers docker.

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
version: "3.7"
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    command: cargo run
    ports:
      - "4000:4000"
    cap_drop:
      - all
    cap_add:
      - NET_BIND_SERVICE
    environment:
      - AWS_ACCESS_KEY_ID=foo
      - AWS_SECRET_ACCESS_KEY=bar
      - AWS_REGION=julia-home
      - AWS_DYNAMODB_ENDPOINT=http://dynamodb:8000
    depends_on:
      - dynamodb
    links:
      - dynamodb
    networks:
      internal_net:
        ipv4_address: 172.21.1.2

  dynamodb:
    container_name: "dynamodb"
    image: amazon/dynamodb-local
    ports:
      - "8000:8000"
    networks:
      internal_net:
        ipv4_address: 172.21.1.1
    environment:
      - ./Djava.library.path=./DynamoDBLocal_lib
    volumes:
      - dynamodata:/home/dynamodblocal/
    working_dir: /home/dynamodblocal/
    command: "-jar DynamoDBLocal.jar -sharedDb -dbPath ."

networks:
  internal_net:
    ipam:
      driver: default
      config:
        - subnet: 172.21.0.0/16

volumes:
  dynamodata:
```

Nosso `docker-compose` precisa de quatro chaves principais: `version`, que corresponde à versão do compose, `services`, que corresponde aos contêineres que vamos rodar, `networks` é a configuração de rede que vamos utilizar, e `volumes` são os volumes compartilhados com os contêineres. A configuração de `networks` simplesmente define uma rede interna com `internal_net` e uma range de subnets em `subnet: 172.21.0.0/16`. Em `services`, precisamos declarar dois contêineres `web`, os quais conterão nossa aplicação e o contêiner `dynamodb`, que conterá a imagem do **DynamoDB**. O contêiner `dynamodb` possui as seguintes chaves:

* `container_name`: é o nome do contêiner, no nosso caso `dynamodb`.
* `image`: a fonte da imagem que estamos utilizando, no caso do DynamoDB é `amazon/dynamodb-local`.
* `ports`: o mapeamento de portas de dentro do contêiner para fora, `8000:8000`.
* `networks`: a definição do IP que vamos utilizar, `ipv4_address: 172.21.1.1`.
* `environment`: configurações de ambiente, `./Djava.library.path=./DynamoDBLocal_lib`, relevante para o dynamo.
* `volumes`: volumes disponíveis para o dynamo utilizar, `dynamodata:/home/dynamodblocal/`.
* `working_dir`: diretório no qual o dynamo executará, `/home/dynamodblocal/`.
* `command`: para inicializar o dynamo `"-jar DynamoDBLocal.jar -sharedDb -dbPath ."`.

Depois disso temos o `web`, que não vou repetir algumas chaves:

* `build`: o contexto de criação da imagem, `context: .`. No caso, estamos passando um dockerfile chamado `Dockerfile` `dockerfile: Dockerfile`.
* `command`: executamos o comando `cargo run` para essa aplicação.
* `cap_drop` e `cap_add`: correspondem a capacidade de um container de remover ou de adicionar capacidades.
* `environment`: para executar o DynamoDB dessa forma precisamos adicionar algumas variáveis de ambiente para que o `client` configure suas credenciais.
    - `AWS_ACCESS_KEY_ID=foo`
    - `AWS_SECRET_ACCESS_KEY=bar`
    - `AWS_REGION=julia-home`
    - `AWS_DYNAMODB_ENDPOINT=http://dynamodb:8000`
* `depends_on`: define a ordem na qual os serviços devem ser inicializados, assim `dynamodb` é inicializado antes de `web`
* `links`: forma legada de fazer com que dois serviços estejam conectados, atualmente bastaria o `networks`, mas coloquei como exemplo. No caso de `links` e `networks` estarem definidos, é preciso que ambos estejam na mesma rede.

Se executarmos `docker-compose up`, veremos que nossos serviços são inicializados, porém, quando fazemos um request, ocorre uma falha de comunicação. Para resolver essa falha, precisamos alterar nosso cliente para que ele se conecte às configurações da rede do `docker-compose`. Para isso, podemos criar um novo cliente e fazer com que o antigo execute somente com a feature `dynamo` ativada:

```rust
// src/todo_api/db/helpers.rs
// ...
#[cfg(feature = "dynamo")]
pub fn client() -> DynamoDbClient {
    DynamoDbClient::new(Region::Custom {
        name: String::from("us-east-1"),
        endpoint: String::from("http://localhost:8000"),
    })
}

#[cfg(not(feature = "dynamo"))]
pub fn client() -> DynamoDbClient {
    DynamoDbClient::new(Region::Custom {
        name: String::from("julia-home"),
        endpoint: String::from("http://dynamodb:8000"),
    })
}
// ...
```

Além disso, precisamos alterar o `bind` de nosso servidor para expor o serviço para fora do container:

```rust
// ...

#[actix_rt::main]
async fn web_main() -> Result<(), std::io::Error> {    
    HttpServer::new(|| {
        App::new()
        .wrap(Logger::new("IP:%a DATETIME:%t REQUEST:\"%r\" STATUS: %s DURATION:%D"))
        .configure(app_routes)
    })
    .workers(num_cpus::get() + 2)
    .bind("0.0.0.0:4000")
    .unwrap()
    .run()
    .await
}

// ...
```

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
    .wrap(DefaultHeaders::new().header("x-request-id", Uuid::new_v4().to_string()))
    .wrap(Logger::new("IP:%a DATETIME:%t REQUEST:\"%r\" STATUS: %s DURATION:%D"))
    .configure(app_routes)
})
// ...
```

Além disso, precisamos definir o header no `Logger`, para isso usamos a chave `X-REQUEST-ID:%{x-request-id}o` após a `DURATION`, pois somente assim o valor de `x-request-id` será logado:

```rust
// ...
HttpServer::new(|| {
    App::new()
    .wrap(DefaultHeaders::new().header("x-request-id", Uuid::new_v4().to_string()))
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
use crate::todo_api::db::helpers::client;

#[derive(Clone)]
pub struct Clients {
    pub dynamo: rusoto_dynamodb::DynamoDbClient,
}

impl Clients {
    pub fn new() -> Self {
        Self { dynamo: client() }
    }
}
```

Agora podemos utilizar a função `data` em `HttpServer` para passar Clients como argumento. Fazemos isso com `Clients::new()`:

```rust
// ...
use todo_api_web::{
    routes::app_routes,
    model::http::Clients,
};

#[actix_rt::main]
async fn web_main() -> Result<(), std::io::Error> {  
    HttpServer::new(|| {
        App::new()
        .data(Clients::new())
        .wrap(DefaultHeaders::new().header("x-request-id", Uuid::new_v4().to_string()))
        .wrap(Logger::new("IP:%a DATETIME:%t REQUEST:\"%r\" STATUS: %s DURATION:%D X-REQUEST-ID:%{x-request-id}o"))
        .configure(app_routes)
    })
    .workers(num_cpus::get() + 2)
    .bind("0.0.0.0:4000")
    .unwrap()
    .run()
    .await
}
// ...
```

Com isso temos `Clients` disponível no nos nossos controllers, para isso adicionamos o estado com `state: web::Data<Clients>`:

```rust
// ...
use crate::{
    // ...
    todo_api_web::model::{
        http::Clients,
        TodoCard, TodoIdResponse, TodoCardsResponse
    }
};


pub async fn create_todo(state: web::Data<Clients>, info: web::Json<TodoCard>) -> impl Responder {
    let todo_card = adapter::todo_json_to_db(info, uuid::Uuid::new_v4());

    match put_todo(state.dynamo.clone(), todo_card) {
        // ...
    }
}

pub async fn show_all_todo(state: web::Data<Clients>) -> impl Responder {
    match get_todos(state.dynamo.clone()) {
        // ...
    }
}
```

Agora precisamos que as funcões `put_todo` e `get_todos` tenham como argumentos um `client :rusoto_dynamo::DynamoDbClient`:

```rust
// ...
use rusoto_dynamodb::{DynamoDbClient, PutItemInput, ScanInput};

#[cfg(not(feature = "dynamo"))]
pub fn put_todo(client: DynamoDbClient, todo_card: TodoCardDb) -> Option<Uuid> {
    use rusoto_dynamodb::DynamoDb;

    let put_item = PutItemInput {
        table_name: TODO_CARD_TABLE.to_string(),
        item: todo_card.clone().into(),
        ..PutItemInput::default()
    };

    match client.put_item(put_item).sync() {
        // ...
    }
}

#[cfg(not(feature = "dynamo"))]
pub fn get_todos(client: DynamoDbClient) -> Option<Vec<TodoCard>> {
    use rusoto_dynamodb::DynamoDb;

    let scan_item = ScanInput {
        limit: Some(100i64),
        table_name: TODO_CARD_TABLE.to_string(),
        ..ScanInput::default()
    };

    match client.scan(scan_item).sync() {
        // ...
    }
}

#[cfg(feature = "dynamo")]
pub fn get_todos(_: DynamoDbClient) -> Option<Vec<TodoCard>> {
    // ...
}

#[cfg(feature = "dynamo")]
pub fn put_todo(_: DynamoDbClient, todo_card: TodoCardDb) -> Option<Uuid> {
    // ...
}
```

Feito isso, devemos adicionar `.data(Clients::new())` a todos os testes de integração, pois esse argumento é esperado nas funções de controller. Um exemplo seria:

```rust
#[actix_rt::test]
    async fn test_todo_cards_with_value() {
        let mut app = test::init_service(
            App::new()
                .data(Clients::new())
                .configure(app_routes)
        ).await;
    
        // ...
        assert_eq!(todo_cards.cards, mock_get_todos());
    }
```

### Serializando o Response

Até o momento estávamos utilizando o formato de criação de `HttpResponse` da seguinte maneira `HttpResponse::Ok().content_type("application/json").body(serde_json::to_string(&struct).expect("Failed to serialize todo cards"))`, mas existe uma forma que pode simplificar nossa vida por nos permitir delegar a chamada de `serde_json`. Esse formato substitui o `.body(...)` por `.json(...)`. A vantagem de se utilizar esse formato é que ele reduz a quantidade de código que nós devemos manter, delegando ao actix essa responsabilidade. Nos capítulos introdutórios do livro, falamos que o actix estava com muita vantagem em relação a outros frameworks nos benchmarks da TechEmpower, porém, no caso de serialização JSON, existem alguns frameworks C/C++ à sua frente, inclusive a crate `hyper`. O Objetivo de `body` é principalmente enviar mensagens sem dados estruturados ou estruturados em outros formatos como Edn.

Com esse pequeno refactor, nossos controllers de `todo` serão modificados para o seguinte formato:

```rust
// src/todo_web_api/controller/todo.rs
// ...
pub async fn create_todo(state: web::Data<Clients>, info: web::Json<TodoCard>) -> impl Responder {
    let todo_card = adapter::todo_json_to_db(info, uuid::Uuid::new_v4());

    match put_todo(state.dynamo.clone(), todo_card) {
        None => {
            error!("Failed to create todo card");
            HttpResponse::BadRequest().body("Failed to create todo card")
        }
        Some(id) => HttpResponse::Created()
            .content_type("application/json")
            .json(TodoIdResponse::new(id))
    }
}

pub async fn show_all_todo(state: web::Data<Clients>) -> impl Responder {
    match get_todos(state.dynamo.clone()) {
        None => {
            error!("Failed to read todo cards");
            HttpResponse::InternalServerError().body("Failed to read todo cards")
        }
        Some(todos) => HttpResponse::Ok().content_type("application/json")
            .json(TodoCardsResponse { cards: todos })
    }
}
```

Com isso, nosso código está pronto para receber novos clientes e nós podemos começar a pensar em autenticação.

[Anterior](./03-get.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](./05-auth.md)