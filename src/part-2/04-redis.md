[Anterior](03-recommendations.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](part-3/00-capa.md)

# Adicionando Caching com Redis

Vamos utilizar como plataforma de caching o banco de dados **Redis** e para isso precisamos disponibilizar um container de redis. Podemos fazer isso adicionando o alvo `redis` em um Makefile. Esse Makefile vai conter um comando para executar o docker com `docker run -p 6379:6379 --name some-redis -d redis`. Inclusive podemos incluir um alvo para executar `cargo run`:

```Makefile
redis:
	docker run -p 6379:6379 --name some-redis -d redis

run:
	cargo run

```

Se executarmos `make redis` vamos obter o seguinte output no terminal:

```sh
docker run -p 6379:6379 --name some-redis -d redis
Unable to find image 'redis:latest' locally
latest: Pulling from library/redis
8559a31e96f4: Pull complete 
85a6a5c53ff0: Pull complete 
b69876b7abed: Pull complete 
a72d84b9df6a: Pull complete 
5ce7b314b19c: Pull complete 
04c4bfb0b023: Pull complete 
Digest: sha256:800f2587bf3376cb01e6307afe599ddce9439deafbd4fb8562829da96085c9c5
Status: Downloaded newer image for redis:latest
0190e745a049650ba4a594b2f379483729fb9b5f01b4b1f7467ef4641772e042
```

## Disponibilizando o Redis Client para as Queries

A primeira coisa que precisamos fazer é adicionar a crate ![`redis`](https://github.com/mitsuhiko/redis-rs) ao seu `Cargo.toml`:

```toml
[dependencies]
actix-web = "2.0.0"
actix-rt = "1.0.0"
juniper = "0.14.2"
serde = { version = "1.0.104", features = ["derive"] }
serde_json = "1.0.44"
serde_derive = "1.0.104"
chrono = "0.4.11"
reqwest = { version = "0.10.4", features = ["blocking", "json"] }
redis = "0.16.0"
```

Com isso podemos começar disponibilizando uma função que retorna o `redis::Client`. Na documentação vemos que basta executarmos `redis::Client::open("redis://127.0.0.1/")?` para obtermos o `Client`, mas por moticos de consistência de código criremos um módulo  `boundaries/redis.rs` que possuirá a função `redis_client` cujo tipo de retorno será um `RedisResult<Client>`:

```rust
use redis::{Client,RedisResult};

pub fn redis_client() -> RedisResult<Client> {
    Ok(redis::Client::open("redis://127.0.0.1/")?)
}
```

Com a função `redis_client` implementada podemos pensar em como vamos incluir o caching no nosso sistema. O local que eu acredito ser mais apropriado é em `resolvers/internal.rs`, pois é o estágio anterior a fazermos o request e funciona bem como um controller também. Assim, na função `recommendations_info` podemos criar a conexão do Redis com `let mut con = redis_client()?.get_connection()?`.  `con` é um tipo mutável pois as operações de `set` e `get`, adicionar e ler respectivamente, exigem mutabilidade. Além disso, para podermos utilizar `get` e `set` precisamos utilizar a diretiva `use redis::Commands;`. 

Outro fato importante é definirmos como vamos querer criar essa estratégia de caching. Sabemos que a função `recommendations_info` recebe 3 argumentos, `departure, origin, destination`, então podemos concluir que estes argumentos são chaves para o caching, porém ainda precisamos definir uma estratégia de tempo. Como sei que as passagens não mudam muito de um dia pro outro, vamos utilizar a atual data como principal chave do caching. Podemos fazer isso utilizando a crate `chrono` e sua função `Utc::today().to_string()` que retorna uma data como `2020-06-18UTC`, e vamos salvar essa informação no valor `today`. Para compormos essas chaves podemos utilizar a seguinte declaração `let redis_key = format!("r{}:{}:{}:{}", &today, &departure, &origin, &destination);`, que combinado retorna `r2020-06-18UTC:2020-07-21:POA:GRU`, o `r` faz referência a `recommendations`.

Com `con` podemos chamar a função `get` com a `redis_key` e aplicar um `match` em seu resultado. Este `get` vai nos retornar um Result que caso seja `Ok` vai retornar o retorno o body do request `http` e caso seja `Err` precisaremos aplicar a função `set` com `set(&redis_key, &recommendations_text)`. Algo como:

```rust
match con.get(&redis_key) {
    Ok(response) => response,
    Err(_) => {
        let _recommendations = recommendations(departure, origin, destination)?.text()?;
        let _: () = con.set(&redis_key, &_recommendations)?;
        _recommendations
    }
};
```

Assim, o resultado deste `match` agora pode ser utilizado em um `let recommendations_text = match ...` que será passado para a função `let recommendations: Recommendations = serde_json::from_str(&recommendations_text)?` e retornar um `Ok(recommendations)`:

```rust
use crate::boundaries::{
    http_out::{best_prices, recommendations},
    redis::redis_client};
use crate::schema::{errors::GenericError, model::web::{best_prices::BestPrices, recommendations::Recommendations}};
use redis::Commands;
use chrono::Utc;

// ...

pub fn recommendations_info(
    departure: String,
    origin: String,
    destination: String,
) -> Result<Recommendations, GenericError> {
    let mut con = redis_client()?.get_connection()?;
    let today = Utc::today().to_string();
    let redis_key = format!("r{}:{}:{}:{}", &today, &departure, &origin, &destination);

    let recommendations_text = match con.get(&redis_key) {
        Ok(response) => response,
        Err(_) => {
            let _recommendations = recommendations(departure, origin, destination)?.text()?;
            let _: () = con.set(&redis_key, &_recommendations)?;
            _recommendations
        }
    };
    
    let recommendations: Recommendations = serde_json::from_str(&recommendations_text)?;

    Ok(recommendations)
}
```

## Aplicando Caching a `best_prices`

Agora para aplicarmos caching em `best_prices_info` podemos copiar a solução de `recommendations_info` modificando as funções para `best_prices` e definir a `redis_key` com a inicial `bp`, `let redis_key = format!("bp{}:{}:{}:{}", &today, &departure, &origin, &destination);`:

```rust
use crate::boundaries::{
    http_out::{best_prices, recommendations},
    redis::redis_client};
use crate::schema::{errors::GenericError, model::web::{best_prices::BestPrices, recommendations::Recommendations}};
use redis::Commands;
use chrono::Utc;

pub fn best_prices_info(
    departure: String,
    origin: String,
    destination: String,
) -> Result<BestPrices, GenericError> {
    let mut con = redis_client()?.get_connection()?;
    let today = Utc::today().to_string();
    let redis_key = format!("bp{}:{}:{}:{}", &today, &departure, &origin, &destination);

    let best_prices_text = match con.get(&redis_key) {
        Ok(response) => response,
        Err(_) => {
            let _best_prices = best_prices(departure, origin, destination)?.text()?;
            let _: () = con.set(&redis_key, &_best_prices)?;
            _best_prices
        }
    };

    let best_prices: BestPrices = serde_json::from_str(&best_prices_text)?;

    Ok(best_prices)
}

pub fn recommendations_info(
    departure: String,
    origin: String,
    destination: String,
) -> Result<Recommendations, GenericError> {
    let mut con = redis_client()?.get_connection()?;
    let today = Utc::today().to_string();
    let redis_key = format!("r{}:{}:{}:{}", &today, &departure, &origin, &destination);

    let recommendations_text = match con.get(&redis_key) {
        Ok(response) => response,
        Err(_) => {
            let _recommendations = recommendations(departure, origin, destination)?.text()?;
            let _: () = con.set(&redis_key, &_recommendations)?;
            _recommendations
        }
    };
    
    let recommendations: Recommendations = serde_json::from_str(&recommendations_text)?;

    Ok(recommendations)
}
```

Em nossos exemplos fizemos caching de mesma data, mas é possível passar chaves que expiram com as funções `set_ex` que recebe a quantidade de segundos até expirar no formato `usize`, `pub fn set_ex<'a, K: ToRedisArgs, V: ToRedisArgs>(key: K, value: V, seconds: usize) -> Self`, e a função `pset_ex` que faz a mesma coisa, mas com milisegundos, `pub fn pset_ex<'a, K: ToRedisArgs, V: ToRedisArgs>(key: K, value: V, milliseconds: usize) -> Self`. Outras funcões interessantes de se olhar são `mset_nx`, `getset`, `getrange`, `setrange`, `persist`, `append`, outras funcões podem ser encontradas em https://docs.rs/redis/0.16.0/redis/struct.Cmd.html. 

Nesta parte aprendemos a utilizar GraphQL com Actix, fazer requests HTTP síncronos e salvar essas informações como caching em uma banco de dados Redis. Com isso podemos começar um frontend com WebAssemby capaz de processar as informações do GraphQL em uma single page app que nos permitirá interagir com as passagens da Latam.

[Anterior](03-recommendations.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](part-3/00-capa.md)