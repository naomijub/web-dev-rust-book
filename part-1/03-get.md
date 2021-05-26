# Obtendo todas as Todo Cards inseridas

Existem muitas abordagens para como vamos adicionar um novo endpoint no nosso sistema, mas a abordagem que eu gostaria de tratar aqui é a de começar de cima para baixo, ou seja, criamos um endpoint `GET` que lé todas as `TodoCard` e nos retorna elas no formato Json. Dessa vez vamos começar escrevendo um teste para este novo endpoint:

```rust
mod read_all_todos {
    use todo_server::todo_api_web::{
        routes::app_routes
    };

    use actix_web::{
        test, App,
        http::StatusCode,
    };
    use actix_service::Service;

    #[actix_rt::test]
    async fn test_todo_index_ok() {
        let mut app = test::init_service(
            App::new()
                .configure(app_routes)
        ).await;
    
        let req = test::TestRequest::with_uri("/api/index").to_request();
    
        let resp = app.call(req).await.unwrap();
        assert_eq!(resp.status(), StatusCode::OK);
    }
}
```

Felizmente, nosso teste falha retornanto um `NOT_FOUND` e nos obriga a implementar a nova rota, `index` em `src/todo_api_web/routes.rs`:

```rust
pub fn app_routes(config: &mut web::ServiceConfig) {
    config.service(
        web::scope("/")
            .service(
                web::scope("api/")
                .route("create", web::post().to(create_todo))
                .route("index", web::get().to(show_all_todo)))
            .route("ping", web::get().to(pong))
            .route("~/ready", web::get().to(readiness))
            .route("", web::get().to(|| HttpResponse::NotFound())),
    );
}
```

Note que agora estamos utilizando uma nova função controller chamada de `show_all_todo`, ela precisa ser incorporada no escopo da função, fazemos isso através de `use crate::todo_api_web::controller::todo::show_all_todo` e recebemos um aviso de que ela não existe, assim devemos implementá-la no módulo `src/todo_api_web/controller/todo.rs`:

```rust
pub async fn show_all_todo() -> impl Responder {
    HttpResponse::Ok()
}
```

Como nosso teste checa apenas o retorno do status `200`, isso é suficiente. Nosso próximo passo é implementar um teste um pouco mais robusto. Esse teste consiste em garantir que o JSON recebido possua um vetor de tamanho 1 após um post em `api/create` ser enviado:

```rust
od read_all_todos {
    use todo_server::todo_api_web::{
        model::TodoCardsResponse,
        routes::app_routes
    };

    use actix_web::{
        test, App,
        http::StatusCode,
    };
    use actix_service::Service;
    use serde_json::from_str;

    use crate::helpers::read_json;

    #[actix_rt::test]
    async fn test_todo_index_ok() {
        // ...
    }

    #[actix_rt::test]
    async fn test_todo_cards_count() {
        let mut app = test::init_service(
            App::new()
                .configure(app_routes)
        ).await;
    
        let post_req = test::TestRequest::post()
            .uri("/api/create")
            .header("Content-Type", "application/json")
            .set_payload(read_json("post_todo.json").as_bytes().to_owned())
            .to_request();

        let _ = app.call(post_req).await.unwrap();
        let req = test::TestRequest::with_uri("/api/index").to_request();
        let resp = test::read_response(&mut app, req).await;

        let todo_cards: TodoCardsResponse = from_str(&String::from_utf8(resp.to_vec()).unwrap()).unwrap();
        assert_eq!(todo_cards.cards.len(), 1);
    }
}
```

Para fazer isso vamos criar uma struct serializável para o formato Json. Essa struct se encontrará em `sr/todo_api_web/model/mod.rs` e se chamará `TodoCardsResponse`:

```rust
#[derive(Serialize, Deserialize)]
pub struct TodoCardsResponse {
    pub cards: Vec<String>
}
```

Note que no momento não precisamos nos preocupar com o tipo de resposta, somente com a struct e seus campos. Agora precisamos fazer nosso controller retornar um vetor com uma String:

```rust
//src/todo_api_web/controller/todo.rs
pub async fn show_all_todo() -> impl Responder {
    HttpResponse::Ok()
        .content_type("application/json")
        .body(serde_json::to_string(&TodoCardsResponse{cards: vec![String::from("test")]}).expect("Failed to serialize todo cards"))
}
```

Com este teste pronto, nosso próximo teste fica bastante simples, pois agora precisamos fazer um teste quase igual, mas que garanta que o retorno seja um `TodoCard` com as informações que postamos. Note que como este teste conterá um mock da resposta do banco de dados, podemos simplesmente adicionar um `Uuid` pré-determinado no mock. Vou criar uma função de teste, no módulo de `helpers` que retorna um vetor com uma `TodoCard`, `mock_get_todos`.

> Note que `TodoCard` não possui um id, assim temos duas opções: a primeira é criar um `TodoCardResponse`, que contém um Id e a segunda é modificarmos a `TodoCard` para conter um campo `id: Option<Uuid>`. Nós vamos seguir a segunda abordagem, cuja única mudança será adicionar `id: None,` no teste `converts_json_to_db` encontrado em `src/todo_api/adapter/mod.rs`.

```rust
// ...
use todo_server::todo_api_web::model::{State, Task, TodoCard};

// ...

pub fn mock_get_todos() -> Vec<TodoCard> {
    vec![TodoCard {
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
    }]
}
```

Com nossa função implementada, podemos criar o novo cenário de teste no submódulo `read_all_todos`:

```rust
#[actix_rt::test]
async fn test_todo_cards_with_value() {
    let mut app = test::init_service(
        App::new()
            .configure(app_routes)
    ).await;

    let post_req = test::TestRequest::post()
        .uri("/api/create")
        .header("Content-Type", "application/json")
        .set_payload(read_json("post_todo.json").as_bytes().to_owned())
        .to_request();

    let _ = app.call(post_req).await.unwrap();
    let req = test::TestRequest::with_uri("/api/index").to_request();
    let resp = test::read_response(&mut app, req).await;

    let todo_cards: TodoCardsResponse = from_str(&String::from_utf8(resp.to_vec()).unwrap()).unwrap();
    assert_eq!(todo_cards.cards, mock_get_todos());
}
```

Veja que agora os tipos de `todo_cards.cards, mock_get_todos()` são incompatíveis, assim, devemos modificar a a struct `TodoCardsResponse` para:

```rust
#[derive(Serialize, Deserialize, PartialEq)]
pub struct TodoCardsResponse {
    pub cards: Vec<TodoCard>,
}
```

Também é necessário, para fins de teste, implementarmos a trait `PartialEq` para todas as structs, e enums, derivadas de `TodoCardsResponse`. Com essa mudança, precisamos modificar a lógica do nosso controller já que agora é necessário que ele busque `TodoCard`s no banco. Faremos isso pela função `get_todos`, que retornará `Vec<TodoCard>`. Caso o `match` retorne, não podemos enviar um erro `500`:

```rust
pub async fn show_all_todo() -> impl Responder {
    match get_todos() {
        None => HttpResponse::InternalServerError().body("Failed to read todo cards"),
        Some(todos) => HttpResponse::Ok()
            .content_type("application/json")
            .body(serde_json::to_string(&TodoCardsResponse{cards: todos}).expect("Failed to serialize todo cards")),
    }
}
```

Agora precisamos implementar a função `get_todos`, mas antes vamos implementar a versão de teste (`feature = dynamo`) da função em `src/todo_api/db/todo.rs`:

```rust
#[cfg(feature = "dynamo")]
pub fn get_todos() -> Option<Vec<TodoCard>> {
    use crate::todo_api_web::model::{State, Task};
    use rusoto_dynamodb::DynamoDb;

    let _ = ScanInput {
        limit: Some(100i64),
        table_name: TODO_CARD_TABLE.to_string(),
        ..ScanInput::default()
    };

    Some(vec![TodoCard {
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
    }])
}
```

Note a presença da struct `ScanInput`. Ela está presente como forma de garantir em teste que a construção dela está coerente. Ao rodarmos o teste (comente o `#[cfg(feature = "dynamo")]`), obtemos sucesso! Agora podemos partir para a leitura da base de dados de fato. Nossa função de `get_todos` vai precisar de algumas mudanças como um `let client = client()` e fazer esse `client` executar um `scan` no banco de dados com o valor de `scan_item`. Em caso de `Err` no `match` retornamos `None` e em caso de sucesso precisamos passar a função por um `adapter` que transforma um `ScanOutput` em um vetor de `TodoCard`:

```rust
#[cfg(not(feature = "dynamo"))]
pub fn get_todos() -> Option<Vec<TodoCard>> {
    use crate::todo_api::db::helpers::client;
    use rusoto_dynamodb::DynamoDb;

    let client = client();
    let scan_item = ScanInput {
        limit: Some(100i64),
        table_name: TODO_CARD_TABLE.to_string(),
        ..ScanInput::default()
    };

    match client.scan(scan_item).sync() {
        Ok(resp) => Some(adapter::scanoutput_to_todocards(resp)),
        Err(_) => None,
    }
}
```

Note que limitamos o `ScanInput` a `100i64`, isso se deve ao fato de que o Dynamo não vai responder mais de 100 itens. Se você precisar de mais, é importante realizar filtros no scan. Antes de implementarmos o `adapter`, seria bom dar uma olhada em como é um `ScanOutput`:

```rust
ScanOutput { 
    consumed_capacity: None, 
    count: Some(2), 
    items: Some([
        {"title": AttributeValue { b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("title"), ss: None }, 
        "description": AttributeValue { b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("descrition"), ss: None }, 
        "owner": AttributeValue { b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("90e700b0-2b9b-4c74-9285-f5fc94764995"), ss: None }, 
        "state": AttributeValue { b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("Done"), ss: None }, 
        "id": AttributeValue { b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("646b670c-bb50-45a4-ba08-3ab684bc4e95"), ss: None }, 
        "tasks": AttributeValue { b: None, bool: None, bs: None, l: Some([
            AttributeValue { b: None, bool: None, bs: None, l: None, m: Some({
                "is_done": AttributeValue { b: None, bool: Some(true), bs: None, l: None, m: None, n: None, ns: None, null: None, s: None, ss: None }, 
                "title": AttributeValue { b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("blob"), ss: None }
            }), 
            n: None, ns: None, null: None, s: None, ss: None }]), m: None, n: None, ns: None, null: None, s: None, ss: None }
        }, 
        {"owner": AttributeValue { b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("90e700b0-2b9b-4c74-9285-f5fc94764995"), ss: None }, 
        "description": AttributeValue { b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("descrition"), ss: None }, 
        "id": AttributeValue { b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("23997c2c-cd10-477f-8838-e88f2f6d7e7d"), ss: None }, 
        "state": AttributeValue { b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("Done"), ss: None }, 
        "title": AttributeValue { b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("title"), ss: None }, 
        "tasks": AttributeValue { b: None, bool: None, bs: None, l: Some([
            AttributeValue { b: None, bool: None, bs: None, l: None, m: Some({
                "title": AttributeValue { b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("blob"), ss: None }, 
                "is_done": AttributeValue { b: None, bool: Some(true), bs: None, l: None, m: None, n: None, ns: None, null: None, s: None, ss: None }}), n: None, ns: None, null: None, s: None, ss: None }
            ]),m: None, n: None, ns: None, null: None, s: None, ss: None }
        }]), 
    last_evaluated_key: None, 
    scanned_count: Some(2) }
```

Agora podemos começar a implementar a função `scanoutput_to_todocards` e, para isso, vamos escrever o primeiro teste com apenas um `items` em `src/todo_api/adapters/mod.rs`:

```rust
#[cfg(test)]
mod scan_to_cards {
    use super::scanoutput_to_todocards;
    use crate::todo_api_web::model::{Task, TodoCard, State};
    use rusoto_dynamodb::ScanOutput;

    fn scan_with_one() -> ScanOutput {
        let mut tasks_hash = std::collections::HashMap::new();
        tasks_hash.insert("title".to_string(), AttributeValue{ b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("blob".to_string()), ss: None });
        tasks_hash.insert("is_done".to_string(), AttributeValue{ b: None, bool: Some(true), bs: None, l: None, m: None, n: None, ns: None, null: None, s: None, ss: None });
        let mut hash = std::collections::HashMap::new();
        hash.insert("title".to_string(), AttributeValue{ b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("title".to_string()), ss: None });
        hash.insert("description".to_string(), AttributeValue{ b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("description".to_string()), ss: None });
        hash.insert("owner".to_string(), AttributeValue{ b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("90e700b0-2b9b-4c74-9285-f5fc94764995".to_string()), ss: None });
        hash.insert("id".to_string(), AttributeValue{ b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("646b670c-bb50-45a4-ba08-3ab684bc4e95".to_string()), ss: None });
        hash.insert("state".to_string(), AttributeValue{ b: None, bool: None, bs: None, l: None, m: None, n: None, ns: None, null: None, s: Some("Done".to_string()), ss: None });
        hash.insert("tasks".to_string(), AttributeValue { b: None, bool: None, bs: None, l: Some(vec![
            AttributeValue { b: None, bool: None, bs: None, l: None, m: Some(tasks_hash), n: None, ns: None, null: None, s: None, ss: None }]), m: None, n: None, ns: None, null: None, s: None, ss: None });

        ScanOutput { 
            consumed_capacity: None, 
            count: Some(1), 
            items: Some(vec![hash]), 
            last_evaluated_key: None, 
            scanned_count: Some(1) }
    }

    #[test]
    fn scanoutput_has_one_item() {
        let scan = scan_with_one();
        let todos = vec![
            TodoCard {
                title: "title".to_string(),
                description: "description".to_string(),
                state: State::Done,
                id: Some(uuid::Uuid::parse_str("646b670c-bb50-45a4-ba08-3ab684bc4e95").unwrap()),
                owner: uuid::Uuid::parse_str("90e700b0-2b9b-4c74-9285-f5fc94764995").unwrap(),
                tasks: vec![
                    Task {
                        is_done: true,
                        title: "blob".to_string()
                    }
                ]
            }
        ];

        assert_eq!(scanoutput_to_todocards(scan), todos)
    }
}
```

Agora podemos finalmente implementar nossa função `scanoutput_to_todocards` para o caso de 1 `items`:

```rust
pub fn scanoutput_to_todocards(scan: ScanOutput) -> Vec<TodoCard> {
    let item = scan.items.unwrap()[0].to_owned();

    vec![TodoCard {
        id: Some(uuid::Uuid::parse_str(&item.get("id").unwrap().s.clone().unwrap()).unwrap()),
        owner: uuid::Uuid::parse_str(&item.get("owner").unwrap().s.clone().unwrap()).unwrap(),
        title: item.get("title").unwrap().s.clone().unwrap(),
        description: item.get("description").unwrap().s.clone().unwrap(),
        state: State::from(item.get("state").unwrap().s.clone().unwrap()),
        tasks: item .get("tasks").unwrap().l .clone().unwrap()
            .iter()
            .map(|t| Task {
                title: t.clone().m.unwrap().get("title").unwrap()
                    .s.clone().unwrap(),
                is_done: t.clone().m.unwrap().get("is_done").unwrap()
                    .bool.clone().unwrap(),
            })
            .collect::<Vec<Task>>(),
    }]
}
```

Infelizmente o código de `scanoutput_to_todocards` conta com muitas referências e tipos `Option`, o que nos força a ter um excesso de `clone()` e `unwrap()`, mas basicamente estamos navegando por dentro dos tipos de `AttributeValue` e, quando o tipo é um `HashMap`, utilizamos `get`. Agora podemos testar o caso para um `scan` com dois conjuntos de `AttributeValue`. Para isso, vamos isolar a criação dos `HashMap` em `scan_with_one`:

```rust
fn attr_values() -> std::collections::HashMap<String, AttributeValue> {
        let mut tasks_hash = std::collections::HashMap::new();
        tasks_hash.insert(
            "title".to_string(),
            AttributeValue {
                b: None, bool: None, bs: None, l: None, m: None, n: None, 
                ns: None, null: None, s: Some("blob".to_string()), ss: None,
            },
        );
        tasks_hash.insert(
            "is_done".to_string(),
            AttributeValue {
                b: None, bool: Some(true), bs: None, l: None,
                m: None, n: None, ns: None, null: None, s: None,  ss: None,
            },
        );
        let mut hash = std::collections::HashMap::new();
        hash.insert(
            "title".to_string(),
            AttributeValue {
                b: None, bool: None, bs: None, l: None, m: None, n: None,
                ns: None, null: None, s: Some("title".to_string()), ss: None,
            },
        );
        hash.insert(
            "description".to_string(),
            AttributeValue {
                b: None, bool: None, bs: None, l: None, m: None, n: None,
                ns: None, null: None,s: Some("description".to_string()), ss: None,
            },
        );
        hash.insert(
            "owner".to_string(),
            AttributeValue {
                b: None, bool: None, bs: None, l: None, m: None, n: None,
                ns: None, null: None, s: Some("90e700b0-2b9b-4c74-9285-f5fc94764995".to_string()),
                ss: None,
            },
        );
        hash.insert(
            "id".to_string(),
            AttributeValue {
                b: None, bool: None, bs: None, l: None, m: None, n: None,
                ns: None, null: None, s: Some("646b670c-bb50-45a4-ba08-3ab684bc4e95".to_string()),
                ss: None,
            },
        );
        hash.insert(
            "state".to_string(),
            AttributeValue {
                b: None, bool: None, bs: None, l: None, m: None, n: None,
                ns: None, null: None, s: Some("Done".to_string()), ss: None,
            },
        );
        hash.insert(
            "tasks".to_string(),
            AttributeValue {
                b: None, bool: None, bs: None,
                l: Some(vec![AttributeValue {
                    b: None, bool: None, bs: None, l: None,
                    m: Some(tasks_hash), n: None, ns: None,
                    null: None, s: None, ss: None,
                }]),
                m: None, n: None, ns: None, null: None, s: None,  ss: None,
            },
        );
        hash
    }
```

Assim a função `scan_with_one` fica:

```rust
fn scan_with_one() -> ScanOutput {
    let hash = attr_values();

    ScanOutput {
        consumed_capacity: None,
        count: Some(1),
        items: Some(vec![hash]),
        last_evaluated_key: None,
        scanned_count: Some(1),
    }
}
```

E podemos fazer a `scan_with_two` ser:

```rust
fn scan_with_two() -> ScanOutput {
        let hash = attr_values();

        ScanOutput {
            consumed_capacity: None,
            count: Some(2),
            items: Some(vec![hash.clone(), hash]),
            last_evaluated_key: None,
            scanned_count: Some(2),
        }
    }
```

E assim já implementamos o seguinte teste (lembre-se de adicionar a trait `Clone` a `TodoCard` e seus derivados):

```rust
#[test]
fn scanoutput_has_two_items() {
    let scan = scan_with_two();
    let todo = TodoCard {
        title: "title".to_string(),
        description: "description".to_string(),
        state: State::Done,
        id: Some(uuid::Uuid::parse_str("646b670c-bb50-45a4-ba08-3ab684bc4e95").unwrap()),
        owner: uuid::Uuid::parse_str("90e700b0-2b9b-4c74-9285-f5fc94764995").unwrap(),
        tasks: vec![Task {
            is_done: true,
            title: "blob".to_string(),
        }],
    };
    let todos = vec![todo.clone(), todo];

    assert_eq!(scanoutput_to_todocards(scan), todos)
}
```

Nosso teste falha e agora nos permite modificar a função `scanoutput_to_todocards` para retornar um vetor com todos os `TodoCard`s contidos em `ScanOutput`:

```rust
pub fn scanoutput_to_todocards(scan: ScanOutput) -> Vec<TodoCard> {
    scan.items.unwrap()
        .into_iter()
        .map(|item| TodoCard {
            id: Some(uuid::Uuid::parse_str(&item.get("id").unwrap().s.clone().unwrap()).unwrap()),
            owner: uuid::Uuid::parse_str(&item.get("owner").unwrap().s.clone().unwrap()).unwrap(),
            title: item.get("title").unwrap().s.clone().unwrap(),
            description: item.get("description").unwrap().s.clone().unwrap(),
            state: State::from(item.get("state").unwrap().s.clone().unwrap()),
            tasks: item.get("tasks").unwrap().l.clone().unwrap()
                .iter()
                .map(|t| Task {
                    title: t.clone().m.unwrap().get("title")
                        .unwrap().s.clone().unwrap(),
                    is_done: t.clone().m.unwrap().get("is_done")
                        .unwrap().bool.clone().unwrap(),
                })
                .collect::<Vec<Task>>(),
        })
        .collect::<Vec<TodoCard>>()
}
```

A mudança que fizemos é bastante simples. Ela simplesmente consiste em transformar a variável `item` em um argumento da closure de `map`. Dessa forma, scan vira um iterável com `scan.items.unwrap().into_iter()` e, depois do `map`, colecionamos todos os valores com `.collect::<Vec<TodoCard>>()`. Pronto, `adapter` feito. Agora podemos utilizar esse `adapter` na função `get_todos`. Para testar a mudança, podemos executar a aplicação novamente e testar:

![Obtendo todos nossos TodoCards.](../imagens/get_todos.png)

No próximo capítulo, vamos parar um pouco com a criação de endpoints e entender melhor como tornar nosso serviço mais viável para produção
