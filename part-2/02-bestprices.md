# Best Prices

Nesta query vamos fazer uma consulta a uma API externa que retorna os melhores preços para uma rota (data, origem e destino). Chamaremos esta query de `bestPrices` e consultaremos a URL `https://bff.latam.com/ws/proxy/booking-webapp-bff/v1/public/revenue/bestprices/oneway?departure=<YYYY-mm-dd>&origin=<IATA>&destination=<IATA>&cabin=Y&country=BR&language=PT&home=pt_br&adult=1&promoCode=`, na qual `departure` é a data de partida no formato `ano-mes-dia`, `origin` é o código IATA da cidade ou do aeroporto de origem, `destination` é o código IATA da cidade ou do aeroporto de destino. Assim, nossa query deve receber 3 argumentos `departure`, `origin` e `destination`, além de lançar erros, caso estes argumentos não estejam dentro do padrão esperado. Com isso, nosso primeiro passo será implementar a função `bestPrices` que recebe os 3 argumentos e por enquanto retornará uma `String`.

### Implementando a função básica de `bestPrices`

Nosso objetivo agora é fazer nosso GraphQL responder da seguinte forma:

![`bestPrices` basic query](../imagens/bestpricesbasicquery.png)

A query que usamos é:

```graphql
query {
  bestPrices(
    departure: "sdf", 
    origin: "sdf", 
    destinantion: "sdfg")
  ping
}
```

E o valor de retorno é:

```json
{
  "data": {
    "bestPrices": "test",
    "ping": "pong"
  }
}
```

O resultado de uma query GraphQL como a que mostamos retorna o campo `data` que é um mapa contendo os resultados das queries `bestPrices` e `ping`. Para resolvermos isso, podemos escrever a seguinte função:

```rust
// schema/mod.rs
// ...
#[juniper::object]
impl QueryRoot {
    //...
    fn bestPrices(departure: String, origin: String, destinantion: String) -> FieldResult<String> {
         Ok(String::from("test"))
    }
}
// ...
```

Com esta função não temos as validações de argumentos que comentamos anteriormente, por isso, podemos comecar escrevendo estes cenários de teste.