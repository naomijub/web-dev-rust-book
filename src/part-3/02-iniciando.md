[Anterior](01-setup.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](03-best-prices.md)

# Iniciando o projeto

Para este passo vamos criar um novo projeto chamado `wasm-airline` com o comando `cargo new --lib wasm-airline`. Em nosso `Cargo.toml` vamos adicionar o seguinte código:

```toml
[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
yew = "0.16"
wasm-bindgen = "0.2"
```

Depois vamos ao nosso arquivo `src/lib.rs` e adicionamos o seguinte código:

```rust
mod app;

use wasm_bindgen::prelude::*;
use yew::prelude::App;


#[wasm_bindgen(start)]
pub fn run_app() {
    App::<app::Airline>::new().mount_to_body();
}
```

E com isso precisamos criar a struct `Airline` no módulo `app`, cuja estrutura base será igual a do exemplo de `hello-world`:

```rust
// src/app.rs
use yew::prelude::*;

pub struct Airline {}

pub enum Msg {}

impl Component for Airline {
    type Message = Msg;
    type Properties = ();

    fn create(_: Self::Properties, _: ComponentLink<Self>) -> Self {
        Airline {}
    }

    fn change(&mut self, _: <Self as yew::html::Component>::Properties) -> bool {
        false
        // Essa função deve retornar true somente se as `Properties` mudarem
     }

    fn update(&mut self, _msg: Self::Message) -> ShouldRender {
        true
    }

    fn view(&self) -> Html {
        html! {
            <p>{ "Hello world!" }</p>
        }
    }
}
```

* esqueça de criar a pasta `static` e incluir o arquivo `static/index.html`:

```html
<html lang="en">
    <head>
        <meta charset="utf-8">
        <title>Yew Sample App</title>
        <script type="module">
            import init from "./wasm.js"
            init()
        </script>
    </head>
    <body></body>
</html>
```

Para executar o projeto vamos criar um `Makefile` que vai executar os comandos que definimos no capítulo anterior para executar o projeto:

```sh
build:
	wasm-pack build --target web --out-name wasm --out-dir ./static

serve:
	miniserve ./static --index index.html

run: build serve
```

Ao executar `make run` podemos acessar `localhost:8080` e encontrar o texto `Hello world!`. Se fizermos uma alteração dentro da função `view` para algo como:

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

Podemos recompilar apenas executando `make build` e recarregar a página. No próximo capítulo vamos iniciar carregando os dados de `BestPrices`.

[Anterior](01-setup.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](03-best-prices.md)