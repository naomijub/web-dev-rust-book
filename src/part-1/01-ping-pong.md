

# Configurando os primeiros endpoints

O nosso objetivo inicial é fazer nosso servidor responder `pong` na rota `/ping` e executar uma função de baixo custo na rota `/~/ready` retornando 200 ou algum valor de status superior a 400, bastante simples. O objetivo disso é definir em nosso servidor endpoints que respondam se o servidor está disponível e saudável, `/ping`, assim como responder se esta pronto para executar mais uma operação, `/~/ready`. Para isso, precisamos criar nosso `todo-server` com o cargo rodando o comando `cargo new todo-server --bin`, que irá gerar os arquivos a seguir:

1. todo-server/src/main.rs
2. todo-server/Cargo.lock 
3. todo-server/Cargo.toml

O arquivo `main.rs` é bastante simples, pois possui somente uma linha executando uma impressão no console de `"Hello, world!"` da seguinte forma:

```rust
fn main() {
    println!("Hello, world!");
}
```

Já o arquivo `Cargo.toml` possui todas as informações sobre o binário gerado:

```toml
[package]
name = "todo-server"
version = "0.1.0"
authors = ["Julia Naomi @naomijub"]
edition = "2018"

[dependencies]
```

Já o arquivo `Cargo.lock` corresponde às configurações geradas para o `Cargo.toml` com o registro de versões de dependências, assim como o `package-lock.json` no Node.

## Adicionando Actix

Nossa principal dependência é o `Actix`, assim precisamos adicionar a dependência `actix-web = "2.0"` à seção `[dependencies]` do `Cargo.toml`:

```toml
[package]
name = "todo-server"
version = "0.1.0"
authors = ["Julia Naomi @naomijub"]
edition = "2018"

[dependencies]
actix-web = "2.0"
```

### Implementando o endpoint `/ping`

Este endpoint é comum a muitos serviços, mas em alguns casos é chamado de `/healthy`, `/healthcheck` ou `/~/healthy`, digamos que seja um exemplo com aplicações práticas de um `hello world`. Neste primeiro momento vamos apresentar primeiro a implementção do `/ping` e depois explicar, pois acredito que neste caso seja importante ter visão do todo antes de entrar nos detalhes. Assim, uma implementação bem simples de `/ping` seria:

 ```rust
use actix_web::{HttpServer, App};
use actix_web::{get, Responder, HttpResponse};

#[get("/ping")]
pub async fn ping() -> impl Responder {
    HttpResponse::Ok().body("pong")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(||{
        App::new()
            .service(healthcheck) 
            .service(ping)
            .default_service(web::to(||HttpResponse::NotFound()))    
    })
    .workers(6)
    .bind(("localhost", 4000))
    .unwrap()
    .run()
    .await
}
```

E seu funcionamento seria:

![Endpoint `/ping`](../imagens/pingpong.png)

![Endpoint `Not Found` para a rota `/`](../imagens/notfound.png)

Agora podemos começar a descrever o endpoint `/ping`: 
1. A primeira coisa que vemos é a diretiva `use` associada a lib `actix_web`. Essa diretiva nos permite disponibilizar no nosso código as funções e estruturas de `actix_web` para uso posterior, assim a diretiva `use actix_web::HttpServer` disponibilizaria a estrutura `HttpServer` para usarmos. 
2. Depois vemos a função `async fn ping() -> impl Responder`. Essa função é uma função assíncrona, devido as palavras reservadas `async fn`, cujo nome é `ping`, recebe nenhum argumento `()` e como tipo de resposta implementa a trait `Responder`, que tem como tipo de retorno `Future<Output = Result<Response, Self::Error>>`. A resposta de `ping` é um status code `Ok()` com um `body("pong")`, porém seria possível também implementar com a função `with_status` da trait `Responder`, ficando `"pong".with_status(StatusCode::Ok)`, que seria classificado como um `CustomResponder`, ou um `Responder` customizado.
3. A seguir encontramos a macro `#[actix_web::main]`, que é habilitada por padrão (https://docs.rs/actix-web/latest/actix_web/#crate-features). A função dessa macro é executar qualquer função marcada como `async` no runtime de actix.
4. Agora temos a função de execução `main` como `async fn main() -> std::io::Result<()> `. Assim, essa [macro](https://docs.rs/actix-web/latest/actix_web/attr.main.html) gera o código necessário para que nossa função `main` esteja conforme o padrão de funções `main` do Rust .
5. A linha `HttpServer::new(|| {..})` permite criar um servidor HTTP com uma `application factory`, assim como permite configurar a instância do servidor, como `workers` e `bind`, que veremos a seguir.
6. A linha `App::new().service(..)` é um `application builder` baseado no padrão *builder* para o `App`, que é uma struct correspondente a aplicação do actix-web, seu objetivo é configurar rotas e settings padrões. A função `service` registra um serviço no servidor.
7. A rota do serviço `ping` é definida pela macro `#[get("/ping")]`.
8. O módulo `web` possui uma série de funções auxiliares e e tipos auxiliares para o actix-web.
9. Depois disso, vemos `workers(6)`, uma função de `HttpServer` que define a quantidade de threads trabalhadoras que estarão envolvidas nesse executável. Por padrão, o valor de `workers` é a quantidade de CPUs lógicas disponíveis.
10. Agora temos o `bind`, que recebe o IP e a porta a qual esse servidor se conectará.
11. `run` e `await` para executar o serviço e esperar pelo `async` definido anteriormente.

É importante também implementarmos um teste para `NOT_FOUND`. Esse teste consiste em um request para uma rota que não existe e um status `NOT_FOUND`: 

```rust
#[actix_web::test]
async fn not_found_route() {
    let mut app = test::init_service(
        App::new()
        .service(healthcheck) 
        .service(ping) 
        .default_service(web::to(|| HttpResponse::NotFound()))
    ).await;

    let req = test::TestRequest::with_uri("/crazy-path").to_request();

    let resp = app.call(req).await.unwrap();
    assert_eq!(resp.status(), StatusCode::NOT_FOUND);
}
```

### Implementando o endpoint /~/ready

Este endpoint é comum especialmente em serviços kubernetes e sua execução é via `kubectl`. Usualmente o `kubectl` espera que o processo ocorra via HTTP, TCP-gRPC ou uma execução de comando no contêiner. Para um contexto simples de contêineres, ter esse endpoint permite um monitoramento mais elevado de serviços, como os *Golden Signals* (sinais dourados apresentados pelo Google no livro Engenharia de Confiabilidade de Sites). Assim, ele permite um pouco mais de informações além de saber se o servidor está vivo (`/ping`), já que verifica se o serviço é capaz de realizar um pequeno processo. Outros endpoints comuns para esse tipo de prova são `/readiness` ou `/~/readiness`. O nosso endpoint vai executar um simples `$ echo hello` e retornar `accepted` para um resultado `Ok` e `internal server error` para um resultado `Err`. 

O primeiro passo para essa prova é definir a rota que vamos chamar, no caso `/~/ready`:

```rust
App::new()
    .service(readiness)
    .service(healthcheck)
    .service(ping) 
    .default_service(web::to(|| HttpResponse::NotFound())) 
```

Agora temos que implementar a função `readiness`:

```rust
#[get("/~/ready")]
pub async fn readiness() -> impl Responder {
    let process = std::process::Command::new("sh")
        .arg("-c")
        .arg("echo hello")
        .output();
    match process {
        Ok(_) => HttpResponse::Accepted(),
        Err(_) => HttpResponse::InternalServerError()
    }
}

```

Note que criamos um valor chamado `process` que é um comando executado pela crate de `OS` `std::process::Command`. Para o `readiness` o comando que estamos executando é `sh -c echo hello`, que imprime `hello` no console. Depois disso fazemos pattern matching do resultado e se for `Ok` retornamos um `2XX` ou retornamos `500` para um `Err`.

### Refatorando

Agora que nosso código está funcionando podemos começar a pensar em organizá-lo, já que nosso arquivo `main` está com muitas funções. A ideia é seguir o padrão do framework Phoenix do Elixir, assim vamos separar o código em  3 conjuntos:

1. `main.rs`, que contém todas as informações de configuração do servidor, ou seja, a própria instância do servidor.
2. `todo_api`, que contém todos os módulos responsáveis por lógica e banco de dados.
3. `todo_api_web`, que contém todos os módulos responsáveis pelo gerenciamento do conteúdo web, como views e controllers, no nosso caso somente controllers.

Assim, nossa primeira refatoração seria mover as funcões que implementam a trait `Responder` para um módulo de controller, `src/todo_api_web/controller/mod.rs`:

```rust
use actix_web::{get, Responder, HttpResponse};

#[get("/~/ready")]
pub async fn readiness() -> impl Responder {
    let process = std::process::Command::new("sh")
        .arg("-c")
        .arg("echo hello")
        .output();
    match process {
        Ok(_) => HttpResponse::Accepted(),
        Err(_) => HttpResponse::InternalServerError(),
    }
}

#[get("/ping")]
pub async fn ping() -> impl Responder {
    HttpResponse::Ok().body("pong")
}
```

Além disso, nosso arquivo `main.rs` agora consome nosso módulo:

```rust
pub mod todo_api_web;

use actix_web::{
    web, App, HttpResponse, HttpServer,
};
use todo_api_web::*;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(readiness)
            .service(ping)
            .default_service(web::to(|| HttpResponse::NotFound()))
    })
    .workers(6)
    .bind(("localhost", 4004))
    .unwrap()
    .run()
    .await
}
```

Note a presença do módulo `todo_api_web` declarado como `mod todo_api_web;` e importando as funções `ping` e `readiness` através de `use todo_api_web::controller::{pong, readiness};`. Além disso, na imagem a seguir podemos perceber a presença de um arquivo `lib.rs` no sistema de arquivos, esse arquivo serve para podermos exportar nossos módulos internos para testes de integração. Assim, atualmente o único módulo declarado em `lib.rs` é `pub mod todo_api_web`.

```rust
//src/todo_web_api/mod.rs
pub mod controller;
```

![Sistema de arquivos após a refatoração](../imagens/basicfilesystem.png)


Outro ponto que creio ser interessante rafatorar é dar a capacidade de nosso servidor adaptar o número de `workers` a quantidade de cores lógicos que a máquina hospedeira possui. Por exemplo, minha máquina pessoal possui 4 cores lógicos e decidi usar uma estratégia de leve estresse aos cores que geralmente se resume a `número de cores lógicos + 2`, ela se torna uma opção segura pelo fato de estarmos utilizando `async` no nosso serviço, ou seja, defini 6 `workers`, mas se meu computador possuísse 8 cores lógicos, eu poderia estar utilizando 10 `workers`. Para resolver este problema podemos utilizar uma lib conhecida como `num_cpus`, basta adicionar ela ao `[dependencies]` do Cargo.toml `num_cpus = "1.0"` e substituir em nosso código da seguinte maneira:

```rust
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(readiness)
            .service(ping)
            .default_service(web::to(|| HttpResponse::NotFound()))
    })
    .workers(num_cpus::get() + 2)
    .bind(("localhost", 4004))
    .unwrap()
    .run()
    .await
}
```

## Testando os endpoints

Uma coisa importante antes de continuarmos é criarmos testes para os endpoints implementados, especialmente agora que já aprendemos como funciona o a criação de rotas e controllers. No caso de rotas e controllers é mais eficiente começar com testes de integração, inclusive por já termos implementado as rotas anteriormente. Assim, precisamos criar alguns arquivos para executar nossos testes de integração. O primeiro arquivo que precisamos é o arquivo `lib` dentro de `tests`, `tests/lib.rs`:

```rust
extern crate todo_server;

mod todo_api_web;
```

Além disso, agora precisamos criar o módulo `todo_api_web` com um módulo interno `controller`:

```rust
// tests/todo_api_web/mod.rs
mod controller;
```

Agora podemos começar a criar os testes de controller no arquivo `tests/todo_api_web/controller.rs`. O primeiro teste que vamos escrever é a verificação se o conteúdo de texto da rota `/ping` é `pong`. Para isso, precisamos utilizar um módulo de suporte para testes do actix chamado `actix_web::test` e incorporar como `[dev-dependencies]` duas libs que nos apoiarão no uso de testes, a `bytes = "0.5.3"` para processar os bytes da resposta gerada no endpoint, e a `actix-service = "1.0.5"`, que apoia nos testes para chamar um mock de `App` do actix na rota desejada. Sugiro isolar os testes dos controllers `pong` e `readiness` em um módulo conforme a seguir:

```rust
#[cfg(test)]
mod ping_readiness {
    use todo_server::todo_api_web::controller::{ping};

    use actix_web::{test, App};

    #[actix_web::test]
    async fn test_ping_pong() {
        let mut app = test::init_service(App::new().service(ping)).await;

        let req = test::TestRequest::get().uri("/ping").to_request();
        let resp = test::call_service(&mut app, req).await;
        let result = test::read_body(resp).await;
        assert_eq!(std::str::from_utf8(&result).unwrap(), "pong");
    }
}

```
O teste apresentado possui uma macro de teste diferente do usual no Rust. Em vez de ser `#[test]`, utilizamos uma macro de teste que disponibiliza o runtime de actix com `#[actix_web::test]`. Além disso, note que agora nossa função de teste passa a ser `async` e utilizamos vários `await` dentro do teste.

Agora vamos explicar as partes do teste: `test::init_service` disponibiliza um mock de serviço do Actix que recebe como argumento um tipo `App` com a rota, `/ping"`, e designa a essa rota um controller, `.service(ping)`. Além disso, criamos uma instância de `Request` para teste com `test::TestRequest` utilizando o método `get()` na `uri("/ping")`. Depois disso, a `resp` corresponde a ler a resposta que esse serviço `app` daria para o `Request` `req`. Como a resposta de `read_response` são `bytes`, convertemos para string e comparamos com o resultado esperado pelo endpoint.

Com o teste de `pong` implementado, podemos criar o teste de `readiness`. No teste de `readiness` não nos interessa saber o corpo da resposta, assim a sugestão é somente saber se a execução retornou um status `Accepted`. Para esse teste, vamos utilizar o recurso da crate `actix-service`, que nos possibilita fazer chamadas a um serviço através de `<App>.call(<Request>).await`. Assim podemos utilizar o `call` para retornar uma response, na qual podemos acessar o `status()`. O bloco de testes fica assim:

```rust
#[cfg(test)]
mod ping_readiness {
    use todo_server::todo_api_web::controller::{ping, readiness};

    use actix_web::{test, App, http::StatusCode};

    ...

    #[actix_web::test]
    async fn test_readiness() {
        let mut app = test::init_service(App::new().service(readiness)).await;

        let req = test::TestRequest::get().uri("/~/ready").to_request();
        let resp = test::call_service(&mut app, req).await;
        assert_eq!(resp.status(), StatusCode::ACCEPTED);
    }
}
```

Com tudo testado, o próximo passo é configurar nosso serviço para criar uma tarefa todo.

* O código deste capítulo está na bibliografia e solicitações de mudança serão bem vindas para manter o código exemplo atualizado.

