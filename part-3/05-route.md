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

Neste livro vamos abordar as estratégias `1` e `2`.


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