[Anterior](./07-ci.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](../part-2/00-capa.md)

# Concluindo o serviço

Falta pouco para termos nosso serviço pronto, pois precisamos implementar um `get` por id e um `update`. O `get` por id não é muito diferente da rota `index`, a única diferença é que vamos passar um parâmetro `id` e chamaremos a rota de `show` e será um método `GET` também. Já o `update` é um pouco diferente pois vamos enviar um corpo Json com as informações para atualizar em uma rota `update` com o método `PUT`. Assim, os endpoints que vamos implementar são:

1. HTTP autenticado em `show/{id}` com o método `GET`.
2. HTTP autenticado em `update/{id}` com o método `PUT` e um body do tipo Json.

## Show por ID

Como já falamos anteriormente, nosso objetivo agora é recuperar um `TodoCard` com base em seu `id` de inserção no banco de dados. Faremos isso utilizando a mesma função que utilizamos na rota `index`, `scan`. Para isso, sabemos que vamos precisar da rota `show/{id}`, como já mencionamos, e vamos precisar retornar um `TodoCard`. Assim, imagino que um bom teste para este cenário seria o seguinte:

```rust

#[cfg(test)]
mod show_by_id {
    use actix_web::{test, App};
    use dotenv::dotenv;
    use todo_server::todo_api_web::model::{
        http::Clients,
        todo::TodoCard,
    };
    use todo_server::todo_api_web::routes::app_routes;
    use serde_json::from_str;
    use crate::helpers::{mock_get_todos};

    #[actix_rt::test]
    async fn test_todo_card_by_id() {
        dotenv().ok();
        let mut app =
            test::init_service(App::new().data(Clients::new()).configure(app_routes)).await;

        let req = test::TestRequest::with_uri("/api/show/544e3675-19f5-4455-9ed9-9ccc577f70fe").to_request();
        let resp = test::read_response(&mut app, req).await;

        let todo_card: TodoCard =
            from_str(&String::from_utf8(resp.to_vec()).unwrap()).unwrap();
        assert_eq!(&todo_card, mock_get_todos().get(0usize).unwrap());
    }
}
```

Este teste consiste em definir um request com um uuid, neste caso aleatório, para a rota `show` com `test::TestRequest::with_uri("/api/show/544e3675-19f5-4455-9ed9-9ccc577f70fe").to_request()`. Com o request em mão, chamamos o serviço para obter uma respose com `test::read_response(&mut app, req).await` e convertemos esta response em um `TodoCard`, `let todo_card: TodoCard = from_str(&String::from_utf8(resp.to_vec()).unwrap()).unwrap()`. Como vamos mockar a resposta de `TodoCard` com o primeiro valor de `mock_get_todos`, basta comparar os dois com `assert_eq!(&todo_card, mock_get_todos().get(0usize).unwrap())`.

O primeiro passo para resolver este teste é adicionar a rota a função `app_routes`:

```rust
// src/todo_api_web/routes.rs
// ...
pub fn app_routes(config: &mut web::ServiceConfig) {
    config.service(
        web::scope("/")
            .service(
                web::scope("api/")
                    .route("create", web::post().to(create_todo))
                    .route("index", web::get().to(show_all_todo))
                    .route("show/{id}", web::get().to(show_by_id)),
            )
            .service(
                web::scope("auth/")
                    .route("signup", web::post().to(signup_user))
                    .route("login", web::post().to(login))
                    .route("logout", web::delete().to(logout)),
            )
            .route("ping", web::get().to(pong))
            .route("~/ready", web::get().to(readiness))
            .route("", web::get().to(|| HttpResponse::NotFound())),
    );
}
```

Para recebermos o ID como argumento de rota precisamos definir-lo como `{id}`, depois disso fazemos um `GET` redirecionando o request para o controller `show_by_id`:

```rust
// src/todo_api_web/controller/todo.rs
// ...
pub async fn show_by_id(id: web::Path<String>, state: web::Data<Clients>) -> impl Responder {
    let uuid = id.to_string();

    match get_todo_by_id(uuid, state.dynamo.clone()) {
        None => {
            error!("Failed to read todo cards");
            HttpResponse::NotFound().finish()
        }
        Some(todo_id) => HttpResponse::Ok().content_type("application/json")
            .json(todo_id)
    }
}
```

Na função `show_by_id` vemos um ítem novo logo de cara, `web::Path<String>`, a função deste ítem é extrair o conteúdo dos argumentos presentes na url do request, ou seja, todas as chaves encontradas entres os símbolos `{` e `}`, no nosso caso `{id}`. Para o caso de um único argumento a estrutura de `web::Path` é como estamos utilizando, mas para o caso de mais argumentos se utiliza tuplas para definir a sequencia de argumentos, por exemplo `/api/show/{id}/task/{title}`, uma rota para obter o status de uma `task` de um `TodoCard` de `id` específico, obteriamos os valores com `web::Path<(String,String)>`. Valores diferentes de string podem ser passados desde que sejam serializáveis pelo serviço, por exemplo o código que escrevemos poderia substituir `String` por `Uuid`, caso fossemos utiliza-la:

```rust
pub async fn show_by_id(id: web::Path<uuid::Uuid>, state: web::Data<Clients>) -> impl Responder {
    let uuid = id.into_inner().to_string();
    // ...
}
```

Não vamos utilizar o `web::Path` com `Uuid` pois, no futuro, vamos querer enviar um response `BadRequest` caso o campo `id` não seja um `Uuid`. Se deixassemos assim o response seria `InternalServerError`, que não é um status muito indicativo. Mantendo o `web::Path` como `String` passamos ao próximo ítem, uma funcnao de `todo_api/db/todo.rs` que recupera um `TodoCard` com base em seu `id`, `get_todo_by_id`. Os argumentos passados a `get_todo_by_id` são uma `String` contendo o `id` e o cliente para `dynamo`. Essa função retorna o tipo `Option<TodoCard>`, que para o padrão `None` vai retornar um status `NotFound`, indicando que este elemento não foi encontrado e para o caso `Some` vai retornar um `Ok` com um corpo contendo um Json com o valor do `TodoCard` encontrado.

A função `get_todo_by_id` é semelhante a função `get_todos`, mas com uma pequerna diferença, a struct `ScanInput` utilizanda para fazer a busca no banco possui dois campos extras `filter_expression` e `expression_attribute_values`. `filter_expression` é responsável por definir qual vai ser o filtro aplicado a este `scan`, por exemplo `=, >=, <`. No nosso caso, nossa `filter_expression` será `Some("id = :id".into())`, ou seja, vamos procurar um `id` que seja igual ao argumento `:id`. Poderiamos ter mais filtros em `filter_expression`, mas usaremos somente esse. Agora precisamos definir o argumento `:id` para aplicar em `filter_expression`. Este argumento é adicionado a query através de `expression_attribute_values`, que recebe um `HashMap` contendo o nome das chaves, `:id` no nosso caso, e um `AttributeValue` com a informação de `id`:

```rust
use std::collections::HashMap;
use rusoto_dynamodb::{AttributeValue, DynamoDb};


let mut _map = HashMap::new();
let mut attr = AttributeValue::default();
attr.s = Some(id);
_map.insert(String::from(":id"), attr);

let scan_item = ScanInput {
    // ...
    filter_expression: Some("id = :id".into()),
    expression_attribute_values: Some(_map),
    ..ScanInput::default()
}
```

> **Filter Expression**
>
> A lista de possíveis operadores para `filter_expression` é a seguinte:
> * Funções: `attribute_exists | attribute_not_exists | attribute_type | contains | begins_with | size`, todas sensitivas a letras maísculas.
> * Operadores de comparação: `= | <> | < | > | <= | >= | BETWEEN | IN`
> *  Operadores lógicos: `AND | OR | NOT`

Com a Struct `ScanInput` definida podemos executar a query em si com `client.scan(scan_item).sync()` e aplicar um `match` a resposta de `scan`. Existem dois padrões possíveis `Ok` e `Err`, como nosso controller espera um `Option<TodoCard>` retornamos um `None` no caso de `Err`. E no caso de `Ok` ainda temos que cuidar o caso de a resposta de `Ok` vir vazia:

```rust
match client.scan(scan_item).sync() {
    Ok(resp) => {
        let todo_id = adapter::scanoutput_to_todocards(resp);
        if todo_id.first().is_some() {
            debug!("Scanned {:?} todo cards", todo_id);
            Some(todo_id.first().unwrap().to_owned())
        } else {
            error!("Could find todocard with ID.");
            None
        }
    }
    Err(e) => {
        error!("Could not scan todocard due to error {:?}", e);
        None
    }
}
```

Como a estrutura de `resp` é um `ScanOutput`, como em `get_todos`, podemos aplicar o mesmo adapter `adapter::scanoutput_to_todocards` a `resp`, porém a resposta deste adapter será um vetor de `TodoCard`. Como queremos somente um único elemento na resposta dessa query, aplicamos a função `first` e validamos o caso de ela não retornar `Some`, indicando com uma respostas `None`. Para o caso de retornar sim, retornamos um `Option` com o primeiro `TodoCard` com `Some(todo_id.first().unwrap().to_owned())`. A função completa ficou como a seguir, funcnao de teste esta logo depois retornando apenas `Some(TodoCard{...})`:

```rust
#[cfg(not(feature = "dbtest"))]
pub fn get_todo_by_id(id: String, client: DynamoDbClient) -> Option<TodoCard> {
    use rusoto_dynamodb::{AttributeValue, DynamoDb};
    use std::collections::HashMap;

    let mut _map = HashMap::new();
    let mut attr = AttributeValue::default();
    attr.s = Some(id);
    _map.insert(String::from(":id"), attr);

    let scan_item = ScanInput {
        limit: Some(100i64),
        table_name: TODO_CARD_TABLE.to_string(),
        filter_expression: Some("id = :id".into()),
        expression_attribute_values: Some(_map),
        ..ScanInput::default()
    };

    match client.scan(scan_item).sync() {
        Ok(resp) => {
            let todo_id = adapter::scanoutput_to_todocards(resp);
            if todo_id.first().is_some() {
                debug!("Scanned {:?} todo cards", todo_id);
                Some(todo_id.first().unwrap().to_owned())
            } else {
                error!("Could find todocard with ID.");
                None
            }
        }
        Err(e) => {
            error!("Could not scan todocard due to error {:?}", e);
            None
        }
    }
}

#[cfg(feature = "dbtest")]
pub fn get_todo_by_id(id: String, client: DynamoDbClient) -> Option<TodoCard> {
    use rusoto_dynamodb::{AttributeValue, DynamoDb};
    use std::collections::HashMap;
    use crate::todo_api_web::model::todo::{State, Task};

    let mut _map = HashMap::new();
    let mut attr = AttributeValue::default();
    attr.s = Some(id);
    _map.insert(String::from(":id"), attr);

    let scan_item = ScanInput {
        limit: Some(100i64),
        table_name: TODO_CARD_TABLE.to_string(),
        filter_expression: Some("id = :id".into()),
        expression_attribute_values: Some(_map),
        ..ScanInput::default()
    };

    Some(
        TodoCard {
            id: Some(uuid::Uuid::parse_str("be75c4d8-5241-4f1c-8e85-ff380c041664").unwrap()),
            title: String::from("This is a card"),
            description: String::from("This is the description of the card"),
            owner: uuid::Uuid::parse_str("ae75c4d8-5241-4f1c-8e85-ff380c041442").unwrap(),
            tasks: vec![
                Task {
                    title: String::from("title 1"),
                    is_done: true,
                },
                Task {
                    title: String::from("title 2"),
                    is_done: true,
                },
                Task {
                    title: String::from("title 3"),
                    is_done: false,
                },
            ],
            state: State::Doing,
        }
    )
}
```

### Validando o Uuid

Nosso próximo passo é validar que o formato enviado é um `Uuid`. Para isso criaremos um teste que faz um request com um formato aleatório de dado e retorna `BadRequest` com a mesagem que "id deve ser um Uuid".

```rust
#[actix_rt::test]
async fn test_todo_card_without_uuid() {
    dotenv().ok();
    let mut app =
        test::init_service(App::new().data(Clients::new()).configure(app_routes)).await;

    let req = test::TestRequest::with_uri("/api/show/fake-uuid").to_request();
    let resp = test::read_response(&mut app, req).await;

    let message = String::from_utf8(resp.to_vec()).unwrap();
    assert_eq!(&message, "Id must be a Uuid::V4");
}
```

Para resolver este teste a implementação de código é bastante simples, basta adicioanrmos um `if` que verifica se o `parse_str` é do tipo `Err` e em caso de `true` retornar `HttpResponse::BadRequest().body("Id must be a Uuid::V4")`. Assim, nossa função ficou da seguinte forma:

```rust
pub async fn show_by_id(id: web::Path<String>, state: web::Data<Clients>) -> impl Responder {
    let uuid = id.to_string();

    if uuid::Uuid::parse_str(&uuid).is_err() {
        return HttpResponse::BadRequest().body("Id must be a Uuid::V4");
    }

    match get_todo_by_id(uuid, state.dynamo.clone()) {
        None => {
            error!("Failed to read todo cards");
            HttpResponse::NotFound().finish()
        }
        Some(todo_id) => HttpResponse::Ok().content_type("application/json")
            .json(todo_id)
    }
}
```

## Atualizando TodoCards

Agora vamos aprender como atualizar as informações de uma `TodoCard` no DynamoDB. Vamos focar em atualizar somente dois atributos `description` e `state`, depois discutiremos estratégias para implementar updates em `tasks`, pois os outros argumentos são essencialmente iguais a `description` e `state`. Agora precisamos definir como será nosso endpoint de atualização, para isso podemos definir sua rota como `/api/update/{id}` e responderá via método `PUT`. Assim, nosso body conterá os campos `state` e/ou `description`, como no exemplo de `put_todo.json`:

```json
{
	"state": "Doing",
	"description": "dfwgferf"
}
```

Um teste para esse cenário seria o seguinte:

```rust
// tests/test_api_web/controller.rs
// ...
#[cfg(test)]
mod update {
    use actix_web::{test, App, http::StatusCode};
    use dotenv::dotenv;
    use todo_server::todo_api_web::model::{
        http::Clients,
    };
    use todo_server::todo_api_web::routes::app_routes;
    use crate::helpers::{read_json};


    #[actix_rt::test]
    async fn test_todo_card_by_id() {
        dotenv().ok();
        let mut app =
            test::init_service(App::new().data(Clients::new()).configure(app_routes)).await;

        let req = test::TestRequest::put()
            .uri("/api/update/544e3675-19f5-4455-9ed9-9ccc577f70fe")
            .header("Content-Type", "application/json")
            .set_payload(read_json("put_todo.json").as_bytes().to_owned())
            .to_request();

        let resp = test::call_service(&mut app, req).await;
        assert_eq!(resp.status(), StatusCode::OK);
    }
}
```

### Criando a Rota

Temos nosso teste, mas agora precisamos criar a rota em `src/todo_api_web/routes.rs` seguindo o padrão `PUT` na rota `/api/update/{id}`:

```rust
use crate::todo_api_web::controller::{
    // ...
    todo::{create_todo, show_all_todo, show_by_id, update_todo},
};

pub fn app_routes(config: &mut web::ServiceConfig) {
    config.service(
        web::scope("/")
            .service(
                web::scope("api/")
                    .route("create", web::post().to(create_todo))
                    .route("index", web::get().to(show_all_todo))
                    .route("show/{id}", web::get().to(show_by_id))
                    .route("update/{id}", web::put().to(update_todo)),
            )
            // ...
    );
}
```

Agora, precisamos implementar o controller `update_todo` em `src/todo_api_web/controller/todo.rs`:

```rust
pub async fn update_todo(
    id: web::Path<String>,
    info: web::Json<TodoCardUpdate>, 
    state: web::Data<Clients>) -> impl Responder {
    let uuid = id.to_string();

    if uuid::Uuid::parse_str(&uuid).is_err() {
        return HttpResponse::BadRequest().body("Id must be a Uuid::V4");
    }

    match update_todo_info(uuid, info.into_inner(), state.dynamo.clone()) {
        true => HttpResponse::Ok().finish(),
        false => HttpResponse::NotFound().finish()
    }
}
```

Os argumentos para a função `update_todo` são `id` que vem da rota da url `{id}` com `web::Path<String>`, `info` que corresponde ao corpo do `PUT` do tipo `web::Json<TodoCardUpdate>` e o `state` que vem do estao da aplicação com `web::Data<Clients>`. Primeiro passo é converter o campo `id` em `String` com `to_string` para validar se essa string é um `Uuid` com `uuid::Uuid::parse_str(&uuid)` e retornar um `HttpResponse::BadRequest().body("Id must be a Uuid::V4")` caso o resultado de `parse_str` seja do tipo `Err`:

```rust
let uuid = id.to_string();

if uuid::Uuid::parse_str(&uuid).is_err() {
    return HttpResponse::BadRequest().body("Id must be a Uuid::V4");
}
```

Depois disso, chamamos a função `update_todo_info` que retorna um booleano para aplicarmos pattern matching em `true`, retornando `HttpResponse::Ok().finish()`, ou em `false`, retornando `HttpResponse::NotFound().finish()`. A função `update_todo_info` está localizada em `src/todo_api/db/todo.rs` e é bastante extensa:

```rust
#[cfg(not(feature = "dbtest"))]
pub fn update_todo_info(id: String, info: TodoCardUpdate, client: DynamoDbClient) -> bool {
    use rusoto_dynamodb::{AttributeValue, DynamoDb};
    use std::collections::HashMap;

    let expression = adapter::update_expression(&info);
    let attribute_values = adapter::expression_attribute_values(&info);
    let mut _map = HashMap::new();
    let mut attr = AttributeValue::default();
    attr.s = Some(id);
    _map.insert(String::from("id"), attr);

    let update = UpdateItemInput {
        table_name: TODO_CARD_TABLE.to_string(),
        key: _map,
        update_expression: expression,
        expression_attribute_values: attribute_values,
        ..UpdateItemInput::default()
    };

    match client.update_item(update).sync() {
        Ok(_) => true,
        Err(e) => {
            error!("failed due to {:?}", e);
            false
        }
    }
}
```

A primeira coisa que precisamos ressaltar neste código é o `UpdateItemInput`, que é a struct responsável por executar a atualização da `todo` com o `id` enviado na rota. Os campos necessários são `table_name`, que é o nome da tabela, `key` que é um `AttributeValue` com todos os valores de `key`, no nosso caso é somente `id`, `update_expression` que define quais argumentos serão atualizados através do adapter `adapter::update_expression`, `expression_attribute_values` que contém os argumentos para atualizar as informações através do `adapter::expression_attribute_values` que transforma os valores de `TodoCardUpdate` em um `HashMap<String, AttributeValue>`. Assim, para transformar o `id` em um `HashMap<String, AttributeValue>` podemos utilizar a seguinte lógica:

```rust
let mut _map = HashMap::new();
let mut attr = AttributeValue::default();
attr.s = Some(id);
_map.insert(String::from("id"), attr);
```

A função para executar a atualização no Dynamo é `update_item`, lembre-se que após o `sync` o resultado é do tipo `Result`, por isso do `match`. Já os adapter são os seguintes:

```rust
// src/todo_api/adapter/mod.rs
// ...
pub fn update_expression(info: &TodoCardUpdate) -> Option<String> {
    let data = info.clone();
    match (data.description, data.state) {
        (Some(_), Some(_)) => Some(String::from("SET description = :d, state_db = :s")),
        (_, Some(_)) => Some(String::from("SET  state_db = :s")),
        (Some(_), _) => Some(String::from("SET description = :d")),
        _ => None
    }
}

pub fn expression_attribute_values(info: &TodoCardUpdate) -> Option<HashMap<String, AttributeValue>> {
    let data = info.clone();
    match (data.description, data.state) {
        (Some(desc), Some(state)) => {
            let mut _map = HashMap::new();
            let mut attr_d = AttributeValue::default();
            attr_d.s = Some(String::from(desc));
            let mut attr_s = AttributeValue::default();
            attr_s.s = Some(String::from(state.to_string()));
            _map.insert(String::from(":d"), attr_d);
            _map.insert(String::from(":s"), attr_s);
            Some(_map)
        },
        (_, Some(state)) => {
            let mut _map = HashMap::new();
            let mut attr = AttributeValue::default();
            attr.s = Some(String::from(state.to_string()));
            _map.insert(String::from(":s"), attr);
            Some(_map)
        },
        (Some(desc), _) => {
            let mut _map = HashMap::new();
            let mut attr = AttributeValue::default();
            attr.s = Some(String::from(desc));
            _map.insert(String::from(":d"), attr);
            Some(_map)
        },
        _ => None
    }
}
```

`update_expression` é responsável pro criar a expressão que vai determinar o que será atualizado. Como recebemos 2 campos `Optional`, `description` e `state`, temos 4 possibilidades:
1. Ambos existem retorna `"SET description = :d, state_db = :s")`.
2. Somente `state` existe retorna `"SET state_db = :s"`.
3. Somente `description` existe retorna `"SET description = :d"`.
4. Nenhum retorna um `None`.

Os testes para `update_expression` são os seguintes:

```rust
#[cfg(test)]
mod update_expression_test {
    use super::update_expression;
    use crate::todo_api_web::model::todo::{State, TodoCardUpdate};

    #[test]
    fn description_and_state() {
        let todo_update = TodoCardUpdate {description: Some("haiushdusd".to_string()), state: Some(State::Doing)};
        let expected = Some(String::from("SET description = :d, state_db = :s"));

        assert_eq!(expected, update_expression(&todo_update));
    }

    #[test]
    fn description() {
        let todo_update = TodoCardUpdate {description: Some("haiushdusd".to_string()), state: None};
        let expected = Some(String::from("SET description = :d"));

        assert_eq!(expected, update_expression(&todo_update));
    }

    #[test]
    fn state() {
        let todo_update = TodoCardUpdate {description: None, state: Some(State::Doing)};
        let expected = Some(String::from("SET state_db = :s"));

        assert_eq!(expected, update_expression(&todo_update));
    }

    #[test]
    fn none() {
        let todo_update = TodoCardUpdate {description: None, state: None};
        let expected = None;

        assert_eq!(expected, update_expression(&todo_update));
    }
}
```

Já `expression_attribute_values` é um pouco mais complicada pois deve retornar um `Option<HashMap<String, AttributeValue>>`, mas as regras de pattern matching são as mesmas. Assim vamos entender o caso que existe tanto `description` quanto `state`. Para `update_expression` não nos interessava o conteúdo da expression, assim utilizavamos `Some(_)` para fazer pattern matching, porém em `expression_attribute_values` eles interessam já que será inseridos dentro do `HashMap`. A primeira cosia que devemos fazer é criar um `HashMap` com `let mut _map = HashMap::new();` e determinar os `AttributeValue` para `state` e para `description`, `let mut attr_s = AttributeValue::default();` e `let mut attr_d = AttributeValue::default();` respectivamente. Depois disso, inserimos o conteúdo de `state` e de `description` no campo `s`, de String, através de `attr_d.s`, `attr_s.s = Some(String::from(state.to_string()));` e `attr_d.s = Some(String::from(desc));`. Inserimos estes valores no mapa com `_map.insert(String::from(":d"), attr_d); _map.insert(String::from(":s"), attr_s);` e retornamos seu valor em `Some(_map)`. A função para teste é a seguinte: 

```rust
#[cfg(feature = "dbtest")]
pub fn update_todo_info(id: String, info: TodoCardUpdate, client: DynamoDbClient) -> bool {
    use rusoto_dynamodb::{AttributeValue, DynamoDb};
    use std::collections::HashMap;

    let expression = adapter::update_expression(&info);
    let attribute_values = adapter::expression_attribute_values(&info);
    let mut _map = HashMap::new();
    let mut attr = AttributeValue::default();
    attr.s = Some(id);
    _map.insert(String::from("id"), attr);

    let update = UpdateItemInput {
        table_name: TODO_CARD_TABLE.to_string(),
        key: _map,
        update_expression: expression,
        expression_attribute_values: attribute_values,
        ..UpdateItemInput::default()
    };

    true
}
```

Agora vamos entender como nosso código mudaria para incluir os outros campos de atualização.

## Atualizando outros campos

Considerando que a struct que temos no banco de dados é a seguinte e que o campo `id` não será atualizado, podemos discutir como adicionar `title`, `owner` e `tasks`:

```rust
pub struct TodoCard {
    pub id: Option<Uuid>,
    pub title: String,
    pub description: String,
    pub owner: Uuid,
    pub tasks: Vec<Task>,
    pub state: State,
}
```

Bom, `title` e `owner` são bastante triviais, pois bastaria expandir nossos adapters para lidarem com mais duas strings, modificando nossa struct `TodoCardUpdate` para:

```rust
pub struct TodoCardUpdate {
    pub description: Option<String>,
    pub state: Option<State>,
    pub title: Option<String>,
    pub owner: Option<Uuid>
}
```

Já o adapter `update_expression` ficaria semelhante ao seguinte:

```rust
pub fn update_expression(info: &TodoCardUpdate) -> Option<String> {
    let data = info.clone();
    match (data.description, data.state, data.title, data.owner) {
        (Some(_), Some(_), Some(_), Some(_)) => Some(String::from("SET description = :d, state_db = :s, title = :t, owner = :o")),
        ...
        (Some(_), Some(_), _, _) => Some(String::from("SET description = :d, state_db = :s")),
        (_, Some(_), Some(_), _) => Some(String::from("SET title = :t, state_db = :s")),
        (_, _, Some(_), Some(_)) => Some(String::from("SET title = :t, owner = :o")),
        (Some(_), _, _, Some(_)) => Some(String::from("SET description = :d, owner = :o")),
        ...
        (_, Some(_), _, _) => Some(String::from("SET  state_db = :s")),
        (Some(_), _, _, _) => Some(String::from("SET description = :d")),
        (_, _, Some(_), _) => Some(String::from("SET title = :t")),
        (_, _, _, Some(_)) => Some(String::from("SET owner = :o")),
        _ => None
    }
}
```

Acredito que esta solução pode ficar um pouco verbosa, assim, uma ideia seria transformar esses 4 campos em um vetor e iterar nele de forma posicional, o que não geraria uma solução muito elegante também, mas seria muito útil para o caso de `expression_attribute_values`, como o **pseudo código** a seguir:

```rust
// pseudo código
pub fn expression_attribute_values(info: &TodoCardUpdate) -> Option<HashMap<String, AttributeValue>> {
    let data = info.clone();
    let mut _map = HashMap::new();
    let data_vec = vec![data.description, data.state, data.title, data.owner];

    data_vec.iter()
        .map(|i| if i.is_some() {
            let mut attr = AttributeValue::default();
            attr.s = Some(String::from(i));
            attr
        } else {
            None
        })
        .enumerate(|(idx, item)| 
          match idx {
              0 => (":d".to_string(), item),
              1 => (":s".to_string(), item),
              2 => (":t".to_string(), item),
              3 => (":o".to_string(), item),
              _ => ("".to_string(), None)
          })
        .fold(_map,|acc, i| 
          if i.is_some() {
              acc.insert(i.0, i.1)
          };
          acc);
        Some(_map)
}
```

### Tasks

Agora precisamos discutir `tasks`, elas são mais complicadas pois não criamos o conceito de `id` nelas, assim a solução que eu creio ser mais simples para lidar com elas é criar uma struct que contém três argumentos `is_bool`, `previous_text`, `new_text`. O campo `is_bool` é equivalente ao da struct `Task`, já o argumento `previous_text` é o argumento que identifica qual o texto existente de `Task` no banco, e `new_text` é o texto que queremos atualizar. Para entender como ficaria a adição, a atualização e o remoção teremos o seguinte:

* Adicão: `previous_text = None`, `new_text = Some`.
* Atualização: `previous_text = Some`, `new_text = Some`.
* Remoção: `previous_text = Some`, `new_text = None`.

```rust
pub struct TaskUpdate {
    pub is_bool: bool,
    pub previous_text: Option<String>,
    pub new_text: Option<String>,
}
```

Portanto, quando identificarmos que `previous_text` não existe, criamos uma nova `task`, e quando identificarmos que `new_text` não existe, deletamos a `task` com o texto da `previous_text`. Já a atualização filtramos todas as tasks que contém o `previous_text` com `new_text`, assim se ambos são iguais atualizamos somente `is_bool` e em caso de não existir uma task com `previous_text`, simplesmente criamos uma nova `new_text`. Isso poderia ser feito em endpoint que responde a um `POST` em `/api/update/{id}/tasks`. 

Fica como um bom desafio fazer estas mudanças que discutimos aqui antes de seguir para a próxima parte, assim como criar um endpoint de `DELETE`. Nesta parte aprendemos a criar um serviço `REST` com actix que cria e gerencia tarefas via `create`, `update`, `show` e `index`, salvando estas informações em um DynamoDB. Além disso, criamos um middleware de autenticação e endpoints de autenticação, via diesel. Outros middlewares que utilizamos foi o `Logger`, que infelizmente não funciona com `dotenv`, necessária para o `Logger`, e um middleware que cria o header `x-request-id`. Aprendemos a gerenciar o estado da aplicação com `.data()` e a configurar rotas com `.configure()`. Por último, aprendemos a tornar nosso sistema tolerante a falhas e a configurar o docker com todas as dependências.

Agora vamos aprender a utilizar graphql com Actix para fazer um sistema de busca de rotas de voos.

[Anterior](./07-ci.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](../part-2/00-capa.md)