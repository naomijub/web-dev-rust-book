[Anterior](./04-recommendations.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Apêndice](../appendix.md) | [Bibliografia](../bibliografia.md)

# Aplicando um Router

Neste capítulo vamos aprender a utilizar o `Router` da `YewStack`, pois `Routers` em *Single Page Apps* (SPA)manipulam a exibição de entidades diferentes, dependendo da `URL`. Em vez do comportamento padrão de solicitar um recurso remoto diferente quando um link é clicado, o roteador define a URL localmente para apontar para uma rota válida em seu aplicativo. O roteador detecta essa alteração e decide o que renderizar.

Vamos utilizar o `YewRouter`, adicionando `yew-router = "0.13.0"` ao `Cargo.toml`. o `YewRouter` possui alguns elementos centrais que valem mencionar:

* `Route`: Contém uma `String` contendo tudo que aparece após o domínio na `URL`.
* `RouteService`: Comunica com o brrowser para receber e enviar as `Routes`.
* `RouteAgent`: Dono do `RouteService` e é usado para coordenar os updates quando as rotas mudam, tanto dentro da aplicação quanto por eventos do browser.
* `Switch`: É uma trait utilizada na conversão de `Route`.
* `Router`: O `Router` comunica com o `RouteAgent` e automaticamente resolve a `Route` que recebe do `RouteAgent` para uma das implementações do `Switch`.

## Explicando como funciona o Router

`Router` funciona de forma a aplicar um *pattern macthing* na url recebida pelo browser que instancia um tipo que implementa a trait `Switch` para a rota correspondente. Uma coisa importante de salientar é que utilizar tags de referência, `<a href=...></a>`, para a rota que você deseja não irá funcionar imediatamente e, no melhor cenário, irá fazer o servidor recarregar todo o bunde do App. No pior cenário, retorna apenas um código `404 - Not Found`, caso o serviço não esteja bem configurado. Assim,  utilizar `RouteService, RouteAgent, RouterButton, RouterLink` para definir a rota via `history.push_state()` modificará a rota, sem recarregar todo o App novamente.

**Configuração do Servidor**

Para que um link externo funcione com o App, o servidor precisa estar configurado para retornar o `index.html` para qualquer request `GET`, senão o retorno será sempre `404`. Além disso, não pode ser um redirecionamento `3xx` para o `index.html`, pois isso modificará a url no browser causando uma falha de roteamento. Assim, é preciso que a resposta seja um `200` que contém o `index.html`. Uma vez que o conteúdo de `index.html` carregar, resultando no carregamento de todos os assets para iniciar o App, que detectará a rota atual definindo o estado do App para o estado correspondente.

1. O miserve que estamos utilizando já aponta para o diretório `static/` e para o índice  `index.html`, `miniserve ./static --index index.html`.
2. Se você quiser servir seu App do mesmo servidor que sua API está localizada, uma recomendação é definir a rota da API como `/api` e montar seus assets sob a rota `/` fazendo com que ela retorne o conteúdo de `index.html`.
3. É possivel também configurar o webpack dev server para apontar seus arquivos para `index.hmtl`. Mais informações em https://webpack.js.org/configuration/dev-server/#devserverhistoryapifallback.

Neste livro vamos abordar s estratégias `1`, mas caso você prefira a estratégia `2`, poderiamos utilizar o serviço anterior, com a crate `actix-files = "0.2.2"`, para retornar nosso `index.html` executando um `GET` na rota `"/"`. O serviço fica configurado da seguinte forma:

```rust
use actix_files::NamedFile;
use actix_web::{get, middleware, web, App, Error, HttpResponse, HttpServer};

// Caminho para o diretório `static/`
const ASSETS_DIR: &str = "../../static";

async fn serve_index_html() -> Result<NamedFile, Error> {
    const INDEX_HTML: &str = "index.html";
    let index_file = format!("{}/{}", ASSETS_DIR, INDEX_HTML);

    Ok(NamedFile::open(index_file)?)
}

#[actix_rt::main]
async fn main() -> std::io::Result<()> {
    std::env::set_var("RUST_LOG", "actix_server=info,actix_web=info");
    env_logger::init();

    let localhost: &str = "0.0.0.0";
    let port: u16 = 8000;
    let addr = (localhost, port);

    HttpServer::new(move || {
        App::new()
            .wrap(middleware::Logger::default())
            .service(actix_files::Files::new("/", ASSETS_DIR).index_file("index.html"))
            .default_service(web::get().to(serve_index_html))
    })
    .bind(addr)?
    .workers(4)
    .run()
    .await
}
```

### Entendendo as Rotas

Agora vamos entender um exemplo genérico de como configurar rotas, pois isto nos permitirá estender a lógica para o caso que vamos utilizar. A primeira coisa que precisamos fazer é definir um `enum` que seja responsável pelas rotas, no exemplo chamamos de `enum AppRoute`. Esse `enum` deve implementar a trait `Switch`, que vai correlacionar cada rota `to` a um elemente do `enum`, como o exemplo do `Index`. O exemplo do `AppRoute::Index` aponta para a rota `/` por conta da derivação `#[to = "/"]`, o mesmo vale para `Profile(u32)` que aponta para `#[to = "/profile/{id}"]`, na qual `{id}` é um valor do tipo `u32`. O caso de `Forum(ForumRoute),` é um pouco mais complicado, pois `ForumRoute` é uma sub rota associada a outro enum que implementa suas próprias rotas, mas seria possível deestrutura esses valores em uma  struct. Importante também salientar que o *pattern matching* ocorre de forma sequencial, então se `Index` fosse o primeiro, nenhuma das outras rotas aconteceria, pois todas as rotas seriam `"/"`.


```rust
#[derive(Switch, Debug)]
pub enum AppRoute {
    #[to = "/profile/{id}"]
    Profile(u32),
    #[to = "/forum{*:rest}"]
    Forum(ForumRoute),
    #[to = "/"]
    Index,
}

#[derive(Switch, Debug)]
pub enum ForumRoute {
    #[to = "/{subforum}/{thread_slug}"]
    SubForumAndThread{subforum: String, thread_slug: String}
    #[to = "/{subforum}"]
    SubForum{subforum: String}
}

html! {
    <Router<AppRoute, ()>
        render = Router::render(|switch: AppRoute| {
            match switch {
                AppRoute::Profile(id) => html!{<ProfileComponent id = id/>},
                AppRoute::Index => html!{<IndexComponent/>},
                AppRoute::Forum(forum_route) => html!{<ForumComponent route = forum_route/>},
            }
        })
    />
}
```

Por último, a construção do `Router` se da através da tag `Router` que recebe como propridade `<AppRoute, ()>` e `render` que é a implementação da função `Router::render` no enum `AppRouter`.

## Potencializando o componente `Airline`

A ideia agora é tornar nosso componente de `Airline` mais flexível para tirarmos maior proveito do `Router`, nesse sentido vamos lidar com as propriedades `departure, origin, destination` para no componente poder fazer queries customizadas. Para iniciarmos este processo, precisamos criar a `struct Props`, que conterá estes campos em `app.rs`:

```rust
#[derive(Properties, Clone)]
pub struct Props {
    pub departure: String,
    pub origin: String,
    pub destination: String
}
```

Note a presença da macro `Properties`, ela é quem nos permite tornar esses campos utilizáveis como propriedades do componente e o fato de que todas as propriedades são `pub` para poderem ser acessadas de fora durante a declaração do componente. Agora precisamos definir `Props` como o tipo `Properties` da trait `Component`, fazemos isso na implementação da trait:

```rust
impl Component for Airline {
    type Message = Msg;
    type Properties = Props;
    // ...
}
```
Podemos seguir com o próximo passo que é salvar as propriedades no estado de `Airline`, que pode ser feito simplesmente adicionando os campos a struct:

```rust
pub struct Airline {
    // ...
    filter_cabin: String,
    departure: String,
    origin: String,
    destination: String
}
```

Esta etapa nos obriga a salvar as propriedades no estado, podemos fazer isso na função `create` da trait `Component`, que agora terá o nome `props` para o argumento `Self::Properties`, que antes estava como `_: Self::Properties`:

```rust
fn create(props: Self::Properties, link: ComponentLink<Self>) -> Self {
    Airline {
        fetch: FetchService::new(),
        link: link,
        fetch_task: None,
        fetching: true,
        graphql_url: "http://localhost:4000/graphql".to_string(),
        graphql_response: None,
        filter_cabin: String::from("Y"),
        departure: props.departure,
        origin: props.origin,
        destination: props.destination
    }
}
```

Esta alteração nos permite passar como argumento para a função `fetch_gql`, chamada pela função `fetch_data`, os campos `departure, origin, destination`, que nos permitem flexibilizar o request para o servidor GraphQL. Em `fetch_data` a chama de `fetch_gql` passa a ser da sehguinte forma `let request = fetch_gql(self.departure.clone(), self.origin.clone(), self.destination.clone());`. Agora a implementação da função `fetch_gql` no módulo `gql` muda para comportar os novos argumentos:

```rust
pub fn fetch_gql(departure: String, origin: String, destination: String) -> Value {
    json!({
        // ...
    })
}
```

Estes novos argumentos devem ser passados para a `query` através da chave `variables` que conterá um mapa na qual cada chave é o nome da variável que vamos passar para `query`. Além disso, na chave `query` agora devemos adicionar os argumentos de query, fazemos isso colocando `query($departure: String!, $origin: String!, $destination: String!)` antes da primeira chave de abertura, `{`. Perceba que o nome dos campos possui um `$` na frente, que nos permite definir como uma variável para utilizar dentro da `query`, como `recommendations(departure: $departure, origin: $origin, destination: $destination)`. O exemplo a seguir mostra como a modificação fica:

```rust
pub fn fetch_gql(departure: String, origin: String, destination: String) -> Value {
    json!({
        "variables": {
            "departure": departure,
            "origin": origin,
            "destination": destination
        },
        "query": "query($departure: String!, $origin: String!, $destination: String!) {
                recommendations(departure: $departure, 
                    origin: $origin, 
                    destination: $destination) {
                    data{
                    // ...
                    }
                }
                bestPrices(departure: $departure, origin: $origin, destination: $destination) {
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

Para utilizamos esta nova `query`, precisamos passar `Props` como argumento em `run_app`, fazemos isso da seguinte forma:

```rust
#![recursion_limit="1024"]
mod app;
mod gql;
mod best_prices;
mod reccomendation;

use wasm_bindgen::prelude::*;
use yew::prelude::App;
use app::Props;


#[wasm_bindgen(start)]
pub fn run_app() {
    App::<app::Airline>::new().mount_as_body_with_props(Props {
        origin: "POA".to_string(),
        destination: "GRU".to_string(),
        departure: "2020-08-21".to_string(),
    });
}
```

## Criando o `Router`

Para começarmos o sistema de roteamento vamos precisar de um novo componente chamado `Model` que estará localizado no módulo `index`. Assim, a primeira coisa que vamos fazer é definir o `Model` e declarar as rotas no enum `AppRoutes`:

```rust
use yew_router::Switch;
use yew::prelude::*;

#[derive(Switch, Debug, Clone)]
pub enum AppRoute {
    #[to = "/oneway?departure={departure}&origin={origin}&destination={destination}"]
    Oneway {departure: String, origin: String, destination: String},
    #[to = "/"]
    Index
}

#[derive(Debug)]
pub struct Model {}
```

Nosso `AppRoute` possui duas rotas, a primeira é a rota inicial, que renderiza logo que acessamos `localhost:8080`, já a segunda é a rota que navegamos, uma vez que os parâmetros tenham sido definidos. Os parâmetros da rota `Oneway` são os mesmos da struct `Props`, e são definidos na declaração da opção. Note que a url possui o nome dos campos dentro de chaves, `/oneway?departure={departure}&origin={origin}&destination={destination}`, é dessa forma que a macro `Switch` consegue executar as substituições de valores. Precisamos adicioanr ao estado de `Model` os campos de `Oneway`, pois comente assim poderemos altera-los para executar a navegação entre rotas, para isso adicionamos os seguintes campos a `Model`:

```rust
#[derive(Debug)]
pub struct Model {
    route_service: RouteService<()>,
    route: Route<()>,
    link: ComponentLink<Self>,
    origin: String,
    destination: String,
    departure: String
}
```

Os campos `origin, destination, departure` já falamos sobre eles, mas adicionamos outros campos. `link` já mencionamos quando criamos os callbacks para `BestPrices`, mas `route` terá a função de receber uma das rotas que `AppRoute` define, que poderá ser utilizada em pattern matching depois e `route_service` serve para fazer a navegação entre as rotas. Além disso, precisamos da diretiva `use yew_router::{route::Route, service::RouteService, Switch}` para comportar os novos campos. Para começarmos a implementar a trait `Component` vamos precisar de um enum de mensagens, que também vamos chamar de `Msg` e conterá, inicialmente, dois campos:

```rust
pub enum Msg {
    RouteChanged(Route<()>),
    ChangeRoute(AppRoute),
    // ...
}
```

`RouteChanged` será responsável por avisar ao componente que a rota mudou, enquando `ChangeRoute` será responsável por efetivamente mudar a rota. Com isso, podemos iniciar a implementação da trait `Component`:

```rust
impl Component for Model {
    type Message = Msg;
    type Properties = ();

    fn create(_: Self::Properties, link: ComponentLink<Self>) -> Self {
        let mut route_service: RouteService<()> = RouteService::new();
        let route = route_service.get_route();
        let callback = link.callback(Msg::RouteChanged);
        route_service.register_callback(callback);

        Model {
            route_service,
            route,
            link,
            origin: String::new(),
            destination: String::new(),
            departure: String::new() 
        }
    }

    fn update(&mut self, msg: Self::Message) -> ShouldRender {
        match msg {
            Msg::RouteChanged(route) => self.route = route,
            Msg::ChangeRoute(route) => {
                self.route = route.into();
                self.route_service.set_route(&self.route.route, ());
            }
        }
        true
    }

    fn change(&mut self, _: Self::Properties) -> ShouldRender {
        false
    }

    fn view(&self) -> VNode {
        html! {
            <div>
                {
                    "route"
                }
            </div>
        }
    }
}
```

A função `create` começa um pouco diferente que o usual:

```rust
let mut route_service: RouteService<()> = RouteService::new();
let route = route_service.get_route();
let callback = link.callback(Msg::RouteChanged);
route_service.register_callback(callback);
```

Nela declaramos o `route_service` como um `RouteService::new()` e definimos a `route` como o atual estado de `route_service`, `route_service.get_route()`. Depois disso, criamos o `callback` que será chamado quando `Msg::RouteChanged` for enviada para `Model` e registramos esse `callback` em `route_service` com `route_service.register_callback(callback)`.

Nosso `update` vai fazer pattern matching nas duas mensagens, na qual `RouteChanged` atualiza `self.route` e `ChangeRoute` define a rota para navegar com `self.route_service.set_route(&self.route.route, ())`.

```rust
fn update(&mut self, msg: Self::Message) -> ShouldRender {
    match msg {
        Msg::RouteChanged(route) => self.route = route,
        Msg::ChangeRoute(route) => {
            self.route = route.into();
            self.route_service.set_route(&self.route.route, ());
        }
    }
    true
}
```

### Definindo as rotas na `view`

Agora podemos elaborar nossa `view` e para fazermos isso vamos applicar um `match` na rota, `self.route`, através a função `AppRoute::switch` que converte a `Route<()>` em um `Option<AppRoute>`. Para o caso `None`, retornamo `404` com `VNode::from("404")`, para o caso `Some(Index)` vamos apresentar a tela inicial para a pessoa inserir `origin, destination, departure`, para a Rota `Some(AppRoute::Oneway{departure, origin, destination})` vamos criar o componente `Airline` com as propriedades anteriores utilizando `html!{<Airline departure = departure origin = origin destination = destination />},`. As propriedades são passadas no formato `propriedade = valor`.

```rust
fn view(&self) -> VNode {
    html! {
        <div>
            {
                match AppRoute::switch(self.route.clone()) {
                    Some(AppRoute::Index) => self.view_index(),
                    Some(AppRoute::Oneway{departure, origin, destination}) 
                        => html!{<Airline departure = departure origin = origin destination = destination />},
                    None => VNode::from("404")
                }
            }
        </div>
    }
}
```

Agora para a função `self.view_index()`. Ela deve ser implementada em um `impl Model` e conterá o HTML a ser exibido para a rota `Index`:

```rust
impl Model {
    fn change_route(&self, app_route: AppRoute) -> Callback<MouseEvent> {
        self.link.callback(move |_| {
            let route = app_route.clone();
            Msg::ChangeRoute(route)
        })
    }

    fn view_index(&self) -> Html {
        html!{
            <div class="index">
                <div class="row">
                    <div class="input-cell">
                        <p> {"Origin"} </p>
                        <input
                            type = "text",
                            value = &*self.origin,
                            oninput = self.link.callback(|e: InputData| Msg::UpdateOrigin(e.value)),
                        />
                    </div>

                    <div class="input-cell">
                        <p> {"Destination"} </p>
                        <input
                            type = "text",
                            value = &*self.destination,
                            oninput = self.link.callback(|e: InputData| Msg::UpdateDestination(e.value)),
                        />
                    </div>
                </div>

                <div class="row">
                    <div class="input-cell">
                        <p> {"Departure"} </p>
                        <input
                            type = "text",
                            value = &*self.departure,
                            oninput = self.link.callback(|e: InputData| Msg::UpdateDeparture(e.value)),
                        />
                    </div>

                    <div class="input-cell submit">
                        <button onclick=&self.change_route(AppRoute::Oneway
                            {departure: self.departure.clone(), 
                            origin: self.origin.clone(), 
                            destination: self.destination.clone()}) > 
                            {"Submit"}
                        </button>
                    </div>
                </div>
            </div>
        }
    }
}
```

A view está dividida basicamente uma tabela de duas linhas com dois elementos cada. Os 3 primeiro elementos são tags HTML para inserção de texto e farão isso para as propriedades `origin, destination, departure`. A estrutura é bastante simples, definimos o tipo do `input` como `type = "text"`, o valor, `value` como `&self.<propriedade>` e a ação `oninput` como um `callback` para uma mensagem que definimos no padrão `Update<Propriedade>` que recebe o valor de `InputData`. A última célula da tabela é uma tag `button` com a ação de navegar para a nova rota. A ação é `onclick` e recebe a função `change_route` com uma rota `Oneway`.

> A mensagem de Update poderia receber um novo parâmetro como um enum que indicasse qual a propriedade e fazer um pattern matching interno.
> 
> ```rust
> <div class="row">
>                     <div class="input-cell">
>                         <p> {"Origin"} </p>
>                         <input
>                             type = "text",
>                             value = &*self.origin,
>                             oninput = self.link.callback(|e: InputData| Msg::Update(Prop::Origin, e.value)),
>                         />
>                     </div>
> 
>                     <div class="input-cell">
>                         <p> {"Destination"} </p>
>                         <input
>                             type = "text",
>                             value = &*self.destination,
>                             oninput = self.link.callback(|e: InputData| Msg::Update(Prop:Destination, e.value)),
>                         />
>                     </div>
>                 </div>
> 
>                 <div class="row">
>                     <div class="input-cell">
>                         <p> {"Departure"} </p>
>                         <input
>                             type = "text",
>                             value = &*self.departure,
>                             oninput = self.link.callback(|e: InputData| Msg::Update(Prop::Departure, e.value)),
>                         />
>                     </div>
>                 </div>
> ```

`change_route` tem uma execução bastante simples, pois define a nova rota através de `let route: AppRoute = app_route.clone();`, cria a mensagem `Msg::ChangeRoute` com a rota criada e atribui isso a um `callback`. Isso nos força a expandirmos `Msg` para os novos casos que descrevemos:

```rust
pub enum Msg {
    RouteChanged(Route<()>),
    ChangeRoute(AppRoute),
    UpdateOrigin(String),
    UpdateDestination(String),
    UpdateDeparture(String)
}
```

ou

```rust
pub enum Prop {
    Origin,
    Destination,
    Departure
}

pub enum Msg {
    RouteChanged(Route<()>),
    ChangeRoute(AppRoute),
    Update(Prop, String)
}
```

E como última tarefa precisamos adicionar as novas mensagens a função `update`. Tanto `origin`, quanto `destination` limitei fracamente em três caracteres pois códigos IATAs possuem apenas três caracteres.

```rust
fn update(&mut self, msg: Self::Message) -> ShouldRender {
    match msg {
        Msg::RouteChanged(route) => self.route = route,
        Msg::ChangeRoute(route) => {
            self.route = route.into();
            self.route_service.set_route(&self.route.route, ());
        },
        Msg::UpdateOrigin(origin) => self.origin = origin[0..3].to_string(),
        Msg::UpdateDestination(destination) => self.destination = destination[0..3].to_string(),
        Msg::UpdateDeparture(departure) => self.departure = departure
    }
    true
}
```

Seguindo o uso de `Prop`:

```rust
fn update(&mut self, msg: Self::Message) -> ShouldRender {
    match msg {
        Msg::RouteChanged(route) => self.route = route,
        Msg::ChangeRoute(route) => {
            self.route = route.into();
            self.route_service.set_route(&self.route.route, ());
        },
        Msg::Update(prop, value) => match prop {
            Prop::Origin => self.origin = value[0..3].to_string(),
            Prop::Destinationj => self.destination = value[0..3].to_string(),
            Prop::Departure => self.departure = value,
        }
    }
    true
}
```

Para finalizar o css utilizado em `view_index`:

```css

.index {
  display: table;
  height: 20%;
  width: auto;
  background-color: darkblue;
  color: wheat;
  transform: translate(150%, 20%);
}

.row {
  display: table-row;
}

.input-cell {
  display: table-cell;
  padding: 1rem;
}

.submit {
  vertical-align: middle;
  text-align: center;
}
```

E uma imagem com o resultado de `view_index` na rota `"/"`:

![View da rota "/"](../imagens/view_index.png)

Agora basta subir, no diretório do GraphQL, o Redis com `make redis`, subir o GraphQL com `make run` e subir, no diretório wasm, o wasm com `make run` adicionar suas rotas desejada e datas desejadas e procurar seu próximo voo em sua própria API.

Esta parte nos trouxe alguns conceitos de desenvolvimento front-end com Rust e Wasm, nos permitindo consultar a API que criamos na parte anterior. Aprendemos a criar um componente com a trait `COmponent1` que realiza um fetch logo no primeiro render da página, comunicando-se por mensagens que alteram o front-end entre um `loader` e os dados que queremos exibir. Aprendemos a navegar através de um `Router` e `Properties` de um componente ao outro e aprendemos a criar componentes the recebem `inputs` de valores textuais, assim como, componentes que tomam ações com vase em cliques, como o `button`. Além disso, para o serviço que criamos executar com nosso front-end local precisamos utilizar a crate `actix-cors` para configurar o `CORS` da API.

Na API que atendia ao front-end utilizamos a crate `juniper` para configurar o GraphQL em cima do `actix-web`, assim como a crate `reqwest` para realizar requests HTTP para uma API externa e utilizamos a crate `redis` para configurar a comunicação com um container Redis. Já no nosso outro serviço, aprendemos a configurar middlewares, como Logger, para o Actix, sistema de tolerância a falhas com `bastion/fort`, uuids, configurações de json com `serde`, configuramos um DynamoDB com a crate `rusoto`, assim como um `Postgres` com a crate `diesel`, utilizamos a crate `chrono` para gerenciar datas, fizemos sistemas de autenticação que utilizavam `jwt` e `bcrypt` para parsear tokens e senhas, além de  testes extensivos para estes cenários, tanto unitários quanto de integração. Vale mencionar também que configuramos um Travis-CI para este cenário. 

Agora você já pode começar a investir em serviços e front-ends Rust para demonstrar seu poder e levar a palavra do Rust a diante.

[Anterior](./04-recommendations.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Apêndice](../appendix.md) | [Bibliografia](../bibliografia.md)