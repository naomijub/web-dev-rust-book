# Componente de Best Prices

Vamos começar com o código que corresponde ao componente de Best Prices, aquele que é o carrossel de preços na parte da imagem a seguir.

![Imagem do componente de Best Prices](../imagens/bestprices.png)

## Fazendo o request para para nosso serviço da parte 2

A primeira coisa que precisamos fazer para podermos utilizar os dois serviços (frontend e backend) em `localhost` é implementar um sistema de CORS no serviço GraphQL que fizemos anteriormente. Essa mudanca é bastante simples e impacta muito pouco na estrutura do código. O que vamos fazer é adicionar a crate `actix-cors` no `Cargo.toml`:

```toml
[package]
name = "recommendations-gql"
version = "0.1.0"
authors = ["Julia Naomi <jnboeira@outlook.com>"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
actix-web = "2.0.0"
# ...
redis = "0.16.0"
actix-cors = "0.2.0"

[dev-dependencies]
bytes = "0.5.3"
```

Agora no arquivo `main.rs` precisamos adicionar as configurações de CORS, para isso disponibilizamos `Cors` através da diretiva `use actix_cors::Cors;` e dentro da função `main` criamos o recurso de `Cors` envolto em um `wrap` com:

```rust
#[actix_rt::main]
async fn main() -> std::io::Result<()> {
    let resolvers: std::sync::Arc<Resolver> = std::sync::Arc::new(create_resolver());

    HttpServer::new(move || {
        App::new()
            .data(resolvers.clone())
            .wrap(
                Cors::new()
                    // ...
                    .finish(),
            )
            .configure(routes)
            .default_service(web::to(|| async { "404" }))
    })
    .bind("127.0.0.1:4000")?
    .run()
    .await
}
```

Agora precisamos configurar quais domínios, métodos e headers serão adicionados. Existem duas maneiras de fazer isso, a primeira e mais simples é atraveés da função `supported_credentials`:

```rust
App::new()
    .data(resolvers.clone())
    .wrap(
        Cors::new()
            .supports_credentials()
            .max_age(3600)
            .finish(),
    )
    .configure(routes)
    .default_service(web::to(|| async { "404" }))
```

E a segunda é adicionando explicitamente as informações com as funções `allowed_*`. Vamos utilizar esta abordagem:

```rust
App::new()
    .data(resolvers.clone())
    .wrap(
        Cors::new()
            .allowed_origin("http://localhost:8080")
            .allowed_origin("http://127.0.0.1:8080")
            .allowed_origin("http://0.0.0.0:8080")
            .allowed_methods(vec!["GET", "POST"])
            .allowed_headers(vec![header::AUTHORIZATION, header::ACCEPT])
            .allowed_header(header::CONTENT_TYPE)
            .max_age(3600)
            .finish(),
    )
    .configure(routes)
    .default_service(web::to(|| async { "404" }))
```

Agora basta executar um `make redis` e depois um `make run` e esse serviço estará executando.

### Printando o request

O passo mais básico que podemos fazer com um request é printar sua resposta na tela. Não precisamos de nenhum estilo bonito ou organização neste ponto, serve somente para sabermos que o request foi bem sucedido. Para fazermos isso, podemos começar pensando como será nossa `view`. Atualmente nossa `view` está com a seguinte aparência:

```rust
fn view(&self) -> Html {
    html! {
        <div>
            <p>{ "Hello world!" }</p>
            <p>{ "Hello Julia" }</p>
        </div>
    }
}
```

Mas se vamos fazer um request precisaremos de um booleano que indica o estado de `loading` e uma `String` que representa a resposta do do backend GraphQL:

```rust
fn view(&self) -> Html {
    if self.fetching {
        html! {
            <div class="loading">
                {"Loading..."}
            </div>
        } 
    } else {
        html! {
            <div>
                <p>{ 
                    if let Some(data) = &self.graphql_response {
                        data
                    } else {
                        "Failed to fetch"
                    }
                }</p>
            </div>
        }
    }
}
```

Assim, defini as duas propriedades da struct `Airline` são `fetching` que indica que um request está sendo feito e `graphql_response` que corresponde ao corpo da resposta do GraphQL. Nosso `view` possui duas modalidades definidas pelo `if/else`, caso o `self.fetching` seja `true` vamos exibir o texto `Loading...` no HTML, que podemos construir utilizando a macro `html!` e injetando o HTML correspondente os q desejamos executar, no caso `html! {<div class="loading">{"Loading..."}</div>}`. Já no nosso `else` utilizamos a macro `html!` da mesma forma, mas decidimos o que exibir com base no `if-let` que extrai o campo `Option` de  `graphql_response`. Com isso, podemos começar a implementar a struct `Airline` e implementar a função `create` que vai definir os valores iniciais de cada propriedade:

```rust
use yew::prelude::*;
// ...


pub struct Airline {
    // ...
    fetching: bool,
    graphql_response: Option<String>
}

pub enum Msg {
    // ..
    Fetching(bool)
}

impl Component for Airline {
    type Message = Msg;
    type Properties = ();

    fn create(_: Self::Properties, link: ComponentLink<Self>) -> Self {
        Airline {
            // ...
            fetching: false,
            graphql_response: None
        }
    }
    // ...
}
```

Note que a função `create` pertence a trait `Component`, que equivale as funções básicas do React. E o enum `Msg` do tipo `Message` funciona de forma que para cada mensagem enviada uma ação é tomada, o que explica os dois estados do `if/else` reagindo diferentemente ao `self.fetching`, que é recebido pela opção `Msg::Fetching` que recebe um `bool`. Esta funcionalidade de atualizar o estado pertence a função `update` da trait `Component`.  Já a função `update` tem a seguinte aparência:

```rust
impl Component for Airline {
    type Message = Msg;
    type Properties = ();
    // ...
    fn update(&mut self, msg: Self::Message) -> ShouldRender {
        match msg {
            // ...
            Msg::Fetching(fetch) => {
                self.fetching = fetch;
            }
        }
        true
    }
}
```

Na função `update` para cada opção de `Msg` o match toma uma ação. Outra função importante nesse contexto é a função `create` que funciona de forma a atualizar a `view` em caso de alguma propriedade mudar:

```rust
impl Component for Airline {
    type Message = Msg;
    type Properties = ();

    // ...
    fn change(&mut self, _: <Self as yew::html::Component>::Properties) -> bool { 
        false
    }
}
```

Na função `change`, se alguma propriedade mudar, é preciso retornar `true`, e se nada acontecer retornar `false`. Com isso, falamos de todas as funções de implementação obrigatória da trait `Component`, mas ainda temos uma função extra que podemos utilizar para nossa aplicação, a `rendered`, que corresponde ao `componentDidMount` do React, e ela é aplicada de forma diferente para o primeiro `render`:

```rust
fn rendered(&mut self, first_render: bool) {
    if first_render {
        Msg::Fetching(true);
        self.fetch_data(); 
    }
}
```

Para o primeiro `render` atualizamos o estado de `fetching` para true com `Msg::Fetching(true)` para que possamos exibir `Loading...` na `view`:

```rust
fn view(&self) -> Html {
    if self.fetching {
        html! {
            <div class="loading">
                {"Loading..."}
            </div>
        } 
    } 
    // ...
}
```

Depois disso, temos a função que executa o `fetch` em si, `self.fetch_data()`. Para essa função vamos precisar de um novo `impl` que nos permita acessar o `self` de `Airline`:

```rust
use crate::gql::fetch_gql;

impl Airline {
    pub fn fetch_data(&mut self) {
        let request = fetch_gql();
  
          let callback = self.link.callback(
              move |response: Response<Text>| {
                  let (meta, data) = response.into_parts();
                  if meta.status.is_success() {
                      Msg::FetchGql(Some(data))
                  } else {
                      Msg::FetchGql(None)
                  }
              },
          );
  
          let request = Request::builder()
              .method("POST")
              .header("content-type", "application/json")
              .uri(self.graphql_url.clone())
              .body(Json(&request))
              .unwrap();

          let task = self.fetch.fetch(request, callback).unwrap();
          self.fetch_task = Some(task);
          Msg::Fetching(false);
    }
}
```

Primeiro passo de `fetch_data` é função da `fetch_gql` do módulo `gql`, que retorna o Json para executar a query no serviço, que depende da crate `serde_json`:

```rust
use serde_json::{json, Value};

pub fn fetch_gql() -> Value {
    json!({
        "query": "{
             bestPrices(departure: \"2020-07-21\", origin: \"POA\", destination: \"GRU\") {
                bestPrices {
                    date
                    available
                    price {amount}
                }
             }
        }"
    })
}
```
