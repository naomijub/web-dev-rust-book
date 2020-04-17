# Configurando o GraphQL

Agora que temos contexto de como funciona o Actix, será muito mais simples criar um novo serviço, assim o objetivo neste capítulo será focar na parte GraphQL deste novo serviço. Antes vamos entender um pouco o que é GraphQL e porque vamos utlizar essa tecnologia.

## GraphQL

GraphQL é uma tecnologia desenvolvida pelo Facebook que consiste em uma linguagem de queries para APIs e um runtime para executar estas queries. Além disso, GraphQL provê ferramentas para entender e descrever os dados das APIs, da ao cliente o poder de decidir quais dados quer consumir e facilita e evolução de APIs. De forma resumida, são quatro etapas que envolvem o GraphQL:

1. Descrever seus dados via tipos:
```graphql
type Project {
  name: String
  tagline: String
  contributors: [User]
}
```

2. Receber um request com os dados a serem consumidos:
```graphql
{
  project(name: "GraphQL") {
    tagline
  }
}
```

3. Realizar as consultas a todos os serviços/APIs necessários
4. Responder exatamente o que o cliente pediu.
```json
{
    "data": {
        "project": {
            "tagline": "A query language for APIs"
        }
    }
}
```

Assim, o motivo de escolhermos GraphQL como tecnologia para este serviço é a necessidade de consultar diversas fontes para um mesmo request, como mais de uma API e caching.

## Queries Básicas

Vamos começar com o básico, fazer o sistema responder `404 NOT_FOUND` para rotas diversas e depois iniciar com uma query que responderá um simples `pong` quando chamarmos a query `ping`. Para isso, nosso primeiro passo é criar o serviço com `cargo new recommendations-gql --bin`, e adicionar as dependências básicas ao nosso Cargo.toml:

```toml
[dependencies]
actix-web = "2.0.0"
actix-rt = "1.0.0"
juniper = "0.14.2"
serde = { version = "1.0.104", features = ["derive"] }
serde_json = "1.0.44"
serde_derive = "1.0.104"
```

Agora precisamos adicionar o caso de status code `404`. Para esse caso vamos utilizar outro recurso que não utilizamos antes que é o `default_service`. Ele nos permite responder um valor default para qualquer rota não encontrada, neste caso escrevemos `404`:

```rs
// main.rs
use actix_web::{web, App, HttpServer};

#[actix_rt::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(move || {
        App::new()
            .default_service(web::to(|| async { "404" }))
    })
    .bind("127.0.0.1:4000")?
    .run()
    .await
}
```

Pronto! ao acessar `localhost:4000` recebemos um `404`. Próximo passo é adicionarmos a query de `ping`.