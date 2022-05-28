[Anterior](00-capa.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](02-iniciando.md)

# Setup de WebAssembly

Antes de fazermos o setup de WebAssembly para nosso computador, vamos entender o porquê de WebAssembly e o que é YewStack que vamos utilizar.

## WebAssembly

### O que é WebAssembly

WebAssembly, também conhecido como `Wasm`, é um padrão que define um formato binário portável e compacto para executáveis com velocidades quase nativas. Sua correspondente textual da linguagem assembly é o `.wat`, que utiliza `expressões-s`, que lembra um pouco Clojure e Scheme. Sua principal aplicação é como interface simples entre o programa que você desenvolveu e seu host ambiente, assim seu maior objetivo é possibilitar páginas web de alta performance, mas existem outros exemplos aplicando os conceitos de WemAssemlby para mobile por exemplo.

### Por que WebAssembly

Aplicações web JavaScript possui uma certa incerteza quanto a performance que o produto final vai atingir. Seu sistema de tipos dinâmico dificulta muitas otimizações e o Garbage Collector muitas vezes não ajuda por depender de algumas pausas confusas. Além disso, aplicar um superset tipado não garante efetivamente uma melhora nesses quesitos, pois no final das contas, continua tendo que executar JavaScript. Outro grande problema pode ser o fato de pequenas mudanças impactarem profundamente a performance caso você caia em buracos como `fadiga do JavaScript` (JavaScript fatigue) e como `inferno de dependências` (dependency hell), que podem confundir muito o sistema de JIT. Outro ponto importante é o tamanho do código, especialmente quando falamos de *dependency hell*, o JavaScript pode causar grande impacto de performance, pois o tamanho do arquivo que vamos comunicar via internet é crucial.

Assim, Rust provê um controler de baixo nível e uma performance confiável, especialmente por não depender de Garbage Collectors não deterministicos como o JavaScript, por possibilitar arquivos de tamanho reduzido, já que você só recebe o que você precisa, e por dar poder as pessoas que vão desenvolver de como o layout de memória será. Outra vantagem é que WebAssembly permite uma portabilidade muito boa, de forma que você pode começar implementando apenas as partes mais críticas de seu JavaScript.

### YewStack

Yew é um framework moderno para criar apps frontend multi-threaded com WebAssembly e Rust.
* Possui um framework baseado em componentes que permite criar UIs de forma interativa. Assim, quem já possui experiência de React e Elm devem se sentir bastante confortáveis com o funcionamento do Yew.
* Sua performance se da por minimizar chamadas a API de DOM e por permitir muito processamento nos web workers de background.
* Permite ótima interoperabilidade com JavaScript, o que permite desenvolver aproveitando os pacotes NPM e integrar com aplicações JavaScript já existentes.

## Setup

Primeiro devemos começar com o setup do WebAssembly para Rust:

1. Começamos com o `wasm-pack`, uma ferramenta para construir, testar e publicar WebAssembly gerado por Rust. Basta executar o comando `curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh` ou acessar o site https://rustwasm.github.io/wasm-pack/installer/ para mais informações. Outra opção é utilizar o `cargo install wasm-pack`.
2. O segundo passo é instalar o `cargo-generate`, que pode ser feito através do comando `cargo install cargo-generate`. Sua função é facilitar a subida e execução de novos projetos e seus tempaltes em Rust.
3. Por fim, instalar o NPM de sua escolha, uma sugestão é fazer o download pelo site https://www.npmjs.com/get-npm e executar `npm install npm@latest -g`.

### Setup para o Yew

Para utilizarmos o Yew vamos precisar de 2 novos pacotes, `wasm-bindgen` e `cargo-web`, e uma dependência NPM chamada `rollup`. 
1. `wasm-bindgen` pode ser encontrado como `dependencies`. 
2. Para instalar o `cargo-web`  você precisa executar o comando `cargo install cargo-web`. Para o `build` basta utilizar `cargo web build` e o `run` utilizar `cargo web run`. Necessário para `stdweb`.
3. Para o exemplo teste precisamos instalar o `rollup` execute `npm install --global rollup`
4. Para o nosso exemplo de teste vamos utilozar Python, ou Python3 se você preferir, e o módulo `http` isntalado com `pip install http`. Outros servidores também seriam possíveis como o `miniserve` do cargo, que pode ser obtido através do comando `cargo +nightly install miniserve`.


Os alvos suportados são:
* `wasm32-unknown-unknown`
* `wasm32-unknown-emscripten`
* `asmjs-unknown-emscripten`

## Hello World 

Nesse exemplo vamos utilizar o template mínimo da `yew-stack`, para isso faça clone do repositório https://github.com/yewstack/yew-wasm-pack-minimal.git. Você verá que o arquivo `Cargo.toml` está configurado da seguitne forma:

```toml
[package]
authors = [
    "Kelly Thomas Kline <kellytk@sw-e.org>",
    "Samuel Rounce <me@samuelrounce.co.uk>"
]
categories = ["gui", "wasm", "web-programming"]
description = "yew-wasm-pack-minimal demonstrates the minimum code and tooling necessary for a frontend web app with simple deployable artifacts consisting of one HTML file, one JavaScript file, and one WebAssembly file, using Yew, wasm-bindgen, and wasm-pack."
edition = "2018"
keywords = ["yew", "wasm", "wasm-bindgen", "web"]
license = "MIT/Apache-2.0"
name = "yew-wasm-pack-minimal"
readme = "README.md"
repository = "https://github.com/yewstack/yew-wasm-pack-minimal"
version = "0.1.0"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "^0.2"
yew = "0.12"
```

Na tag `author` mude para seu nome e seu email, a `description` pode ser `"hello world in wasm"`, a tag `name` pode ser `hello-world-wasm`, estaremos utilizando `websys`. Caso você deseje utilizar a `stdweb` basta modificar a dependência `yew` para `yew = { version = "0.15", package = "yew-stdweb" }`. Alguns exemplos da diferença entre `websys` e `stdweb`:

```rust
// web-sys
let window: web_sys::Window = web_sys::window().expect("window not available");
window.alert_with_message("hello from wasm!").expect("alert failed");

// stdweb
let window: stdweb::web::Window = stdweb::web::window();
window.alert("hello from wasm!");

// stdweb com a macro js!
use stdweb::js;
use stdweb::unstable::TryFrom;
use stdweb::web::Window;

let window_val: stdweb::Value = js!{ return window; };
let window = Window::try_from(window_val).expect("conversion to window failed");
window.alert("hello from wasm!");
```

Dentro da macro `js!` é possível utilizar JavaScript, como no exemplo `js!{ return window; }`.

**Diferenças entre `web-sys` e `stdweb`**

|  	| web-sys 	| stdweb 	|
|-	|-	|-	|
| Status do projeto 	| Ativamente mantido pelo wasm-working-group 	| Pouca atividade no Github, já passou longos períodos sem commits 	|
| Cobertura da Web API 	| APIs Rust são auto geradas pela spec Web IDL e deveriam ter cobertura de ~100%. | APIs do Browser são adicionadas de acordo com as necessidades da comunidade |
| Design da API Rust 	| Toma medidas conservadoras ao retornar quase sempre um `Result` para as chamadas da API	| Prefere utilizar `panic!` em vez de `Result`.	|
| Suporte para build tools | * `wasm-pack` * `wasm-bindgen` | * `wasm-pack` * `wasm-bindgen` * `cargo-web` |
| Alvos suportados | * `wasm32-unknown-unknown` | * `wasm32-unknown-unknown` * `wasm32-unknown-emscripten` * `asmjs-unknown-emscripten` |

### Executando o projeto

**Opção 1**

1. Para buildar o projeto execute `wasm-pack build --target web`.
2. Para fazer o bundle utilize `rollup ./main.js --format iife --file ./pkg/bundle.js`. 
3. Para expor seu bundle no browser execute o comando `python -m SimpleHTTPServer` para Python2 e `python3 -m http.server` para Python3, depois acesse em seu browser `localhost:8000` para ver um bome  velho `Hello world!`.

**Opção 2**
1. Crie a pasta `static` através de um `mkdir static` e inclua o seguinte arquivo `index.html`:
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
2. Modifique a `src/lib.rs` para expor a função:
```rust
mod app;

use wasm_bindgen::prelude::*;
use yew::App;

#[wasm_bindgen(start)]
pub fn run_app() {
    App::<app::App>::new().mount_to_body();
}
```
3. Execute o comando `wasm-pack build --target web --out-name wasm --out-dir ./static` para buildar seu projeto.
4. Disponibilize o bundle no browser com `miniserve ./static --index index.html`. E pronto, basta acessar `localhost:8080`.

Ao longo das próximas etapas do projeto vamos utilizar a `opção 2`, mas se você preferir utilizar a `opção 1` fique a vontade.

[Anterior](00-capa.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](02-iniciando.md)