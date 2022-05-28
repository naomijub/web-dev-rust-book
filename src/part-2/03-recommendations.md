[Anterior](02-bestprices.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](04-redis.md)

# Recommendations

Nesta query, `recommendations`, vamos fazer uma consulta a mesma API anterior, mas em um endpoint diferente. Esse endpoint retorna todas as opções de voo para uma rota (data, origem e destino). Consultaremos a URL de `recommendations` da Latam `https://bff.latam.com/ws/proxy/booking-webapp-bff/v1/public/revenue/recommendations/oneway?departure=<YYYY-mm-dd>&origin=<IATA>&destination=<IATA>&cabin=Y&country=BR&language=PT&home=pt_br&adult=1&promoCode=`, na qual `departure` é a data de partida no formato `ano-mes-dia`, `origin` é o código IATA da cidade ou do aeroporto de origem, `destination` é o código IATA da cidade ou do aeroporto de destino. Assim, nossa query deve receber 3 argumentos `departure`, `origin` e `destination` e retornar uma lista de todas as opções de vôo, além de lançar erros. Assim como a query `bestPrices`, nossa query de `recommendations` vai receber os mesmos parâmetros:

**BestPrices**:
```rust
bestPrices(
    departure: String,
    origin: String,
    destination: String,
) 
```

**Recommendations**:
```rust
recommendations(
    departure: String,
    origin: String,
    destination: String,
) 
```

O tipo de retorno de `recommendations` será semelhante ao tipo de `bestPrices`, `Result<Recommendations, GenericError>`. As validações dos tipos de entrada podem continuar iguais:

```rust
#[juniper::object]
impl QueryRoot {
    fn ping() -> FieldResult<String> {
        Ok(String::from("pong"))
    }

    fn bestPrices(
        departure: String,
        origin: String,
        destination: String,
    ) -> Result<BestPrices, GenericError> {
        error::iata_format(&origin, &destination)?;
        error::departure_date_format(&departure)?;
        let best_price = best_prices_info(departure, origin, destination)?;
        Ok(best_price)
    }

    fn recommendations(
        departure: String,
        origin: String,
        destination: String,
    ) -> Result<Recommendations, GenericError> {
        error::iata_format(&origin, &destination)?;
        error::departure_date_format(&departure)?;
        // ...
    }
}
```

Agora podemos começar a implementar o resolver `recommendations_info`, começando por conehcer o endpoint.

## Conhecendo o endpoint de `recommendations`

Consultando o endpoint de `recommendations` para data `"2020-07-21"`, para origem `POA` e para destino `GRU` https://bff.latam.com/ws/proxy/booking-webapp-bff/v1/public/revenue/recommendations/oneway?departure={data}&origin={iata}&destination={iata}&cabin=Y&country=BR&language=PT&home=pt_br&adult=1&promoCode= recebemos o seguinte campos relevantes no Json (muitos campos foram ocultos pois não vamos utilizar):

```json
{
  "data":[
    {
      "flights":[
        {
          "flightCode":"LA4596",
          "arrival":{
            "airportCode":"GRU",
            "airportName":"Guarulhos Intl.",
            "cityCode":"SAO",
            "cityName":"São Paulo",
            "countryCode":"BR",
            "date":"2020-07-21",
            "dateTime":"2020-07-21T10:50-03:00",
            "overnights":0,
            "time":{ "stamp":"10:50", "hours":"10", "minutes":"50" }
          },
          "departure":{
            "airportCode":"POA",
            "airportName":"Salgado Filho",
            "cityCode":"POA",
            "cityName":"Porto Alegre",
            "countryCode":"BR",
            "date":"2020-07-21",
            "dateTime":"2020-07-21T09:15-03:00",
            "overnights":null,
            "time":{ "stamp":"09:15", "hours":"09", "minutes":"15" }
          },
          "stops":0,
          "segments":[
            {
              "flightCode":"LA4596",
              "flightNumber":"4596",
              "waitTime":null,
              "equipment":{ "name":"Airbus 321", "code":"321" },
              "legs":[

              ],
              "airline":{
                "marketing":{ "code":"LA", "name":"LATAM Airlines" },
                "operating":{
                  "code":"JJ",
                  "name":"LATAM Airlines Brasil"
                },
                "code":"LA"
              },
              "duration":"PT1H35M",
              "departure":{
                "airportCode":"POA",
                "airportName":"Salgado Filho",
                "cityCode":"POA",
                "cityName":"Porto Alegre",
                "countryCode":"BR",
                "date":"2020-07-21",
                "dateTime":"2020-07-21T09:15-03:00",
                "overnights":null,
                "time":{ "stamp":"09:15", "hours":"09", "minutes":"15" }
              },
              "arrival":{
                "airportCode":"GRU",
                "airportName":"Guarulhos Intl.",
                "cityCode":"SAO",
                "cityName":"São Paulo",
                "countryCode":"BR",
                "date":"2020-07-21",
                "dateTime":"2020-07-21T10:50-03:00",
                "overnights":0,
                "time":{ "stamp":"10:50", "hours":"10", "minutes":"50"
                }
              },
              "familiesMap":{
                "Y-LIGHT":{
                  "segmentClass":"G",
                  "farebasis":"GLYX0N1"
                },
                "Y-PLUS":{
                  "segmentClass":"G",
                  "farebasis":"GLYX0N8"
                },
                "Y-TOP":{
                  "segmentClass":"G",
                  "farebasis":"GLYX0N9"
                },
                "W-PLUS":{
                  "segmentClass":"P",
                  "farebasis":"GDKX0N2"
                },
                "W-TOP":{
                  "segmentClass":"W",
                  "farebasis":"XJ7X0NA"
                }
              }
            }
          ],
          "flightDuration":"PT1H35M",
          "cabins":[
            {
              "code":"Y",
              "displayPrice":101.03,
              "availabilityCount":18,
              "displayPrices":{
                "slice":101.03,
                "wholeTrip":101.03,
                "sliceDiscountCode":"",
                "wholeTripDiscountCode":""
              },
              "fares":[
                {
                  "code":"SL",
                  "category":"LIGHT",
                  "fareId":"250/BdC0XLNrp3H333zqIqmJJpNOO/05D8NwB5zcjHVNnkyl4GjqR/YOQrcDNWLPERAtEBLYqieN002",
                  "price":{
                    "adult":{
                      "amountWithoutTax":68.9,
                      "taxAndFees":32.13,
                      "total":101.03
                    }
                  },
                  "availabilityCount":18,
                },
                {
                  "code":"SE",
                  "category":"PLUS",
                  "fareId":"250/BdC0XLNrp3H333zqIqmJJpNOO/05D8NwB5zcjHVNnkyl4GjqR/YOQrcDNWLPERAtEBLYqieN009",
                  "price":{
                    "adult":{
                      "amountWithoutTax":123.9,
                      "taxAndFees":32.13,
                      "total":156.03
                    }
                  },
                  "availabilityCount":18
                },
                {
                  "code":"SF",
                  "category":"TOP",
                  "fareId":"250/BdC0XLNrp3H333zqIqmJJpNOO/05D8NwB5zcjHVNnkyl4GjqR/YOQrcDNWLPERAtEBLYqieN00C",
                  "price":{
                    "adult":{
                      "amountWithoutTax":193.9,
                      "taxAndFees":32.13,
                      "total":226.03
                    }
                  },
                  "availabilityCount":18
                }
              ]
            },
            {
              "code":"W",
              "displayPrice":247.03,
              "availabilityCount":9,
              "displayPrices":{
                "slice":247.03,
                "wholeTrip":247.03,
                "sliceDiscountCode":"",
                "wholeTripDiscountCode":""
              },
              "fares":[
                {
                  "code":"RA",
                  "category":"PLUS",
                  "fareId":"250/BdC0XLNrp3H333zqIqmJJpNOO/05D8NwB5zcjHVNnkyl4GjqR/YOQrcDNWLPERAtEBLYqieN00F",
                  "price":{
                    "adult":{
                      "amountWithoutTax":214.9,
                      "taxAndFees":32.13,
                      "total":247.03
                    }
                  },
                  "availabilityCount":9,
                },
                {
                  "code":"RY",
                  "category":"TOP",
                  "fareId":"250/BdC0XLNrp3H333zqIqmJJpNOO/05D8NwB5zcjHVNnkyl4GjqR/YOQrcDNWLPERAtEBLYqieN00K",
                  "price":{
                    "adult":{
                      "amountWithoutTax":673.9,
                      "taxAndFees":32.13,
                      "total":706.03
                    }
                  },
                  "availabilityCount":12,
                }
              ]
            }
          ]
        },
        ...Mais recomendações aqui
      ],
      "currency":"BRL",
      "recommendedFlightCode":"LA4596"
    }
  ],
  "status":{
    "code":200,
    "message":""
  }
}
```

Com esse Json podemos começar a modelar a resposta de recommendations, conforme fizemos com `BestPrices`.

### Modelando `Recommendations`

Para modelar recomendarions precisamos fazer uma pequena refatoração em `BestPrices`, para evitarmos a que `Recommendations` e `BestPrices` fiquem misturados no mesmo módulo. Podemos criar um novo arquivo para cada um dos módulos, como temos feito, ou criar módulos internos dentro de `schema/model/web.rs`:

```rust
pub mod best_prices {
    use juniper::GraphQLObject;
    use serde::{Deserialize, Serialize};

    #[derive(Serialize, Deserialize, Debug, PartialEq, Clone, GraphQLObject)]
    #[serde(rename_all = "camelCase")]
    pub struct BestPrices {
        itinerary: Itinerary,
        best_prices: Vec<BestPrice>,
    }

    // ...
}
```

Agora podemos criar o módulo `recommendations` com:

```rust
pub mod recommendations {
    use juniper::GraphQLObject;
    use serde::{Deserialize, Serialize};
    // ...
}
```

Nossa estrutura de `Recommendations` contém 2 campos principais `data` e `status`. `status` é uma struct com os campos `code` do tipo `i32` e `message` do tipo `String`. Já `data` é um vetor de uma struct que contrém os campos `flight`, `currency`, `recommendedFlightCode`, resultando em algo assim:

```rust
pub mod recommendations {
    use juniper::GraphQLObject;
    use serde::{Deserialize, Serialize};

    #[derive(Serialize, Deserialize, Debug, PartialEq, Clone, GraphQLObject)]
    pub struct Recommendations {
        data: Vec<Data>,
        status: Status,
    }

    #[derive(Serialize, Deserialize, Debug, PartialEq, Clone, GraphQLObject)]
    pub struct Status {
        code: i32,
        message: String,
    }

    #[derive(Serialize, Deserialize, Debug, PartialEq, Clone, GraphQLObject)]
    #[serde(rename_all = "camelCase")]
    pub struct Data {
        flights: Vec<Flight>,
        recommended_flight_code: String,
        currency: String,
    }
}
```

Agora precisamos implementar as struct `Flight` derivada do seguinte Json:

```json
{
    "flightCode":"LA4596",
    "arrival":{ ... },
    "departure":{ ... },
    "stops":0,
    "segments":[ ... ],
    "flightDuration":"PT1H35M",
    "cabins":[ ... ]
}
```

Assim, os campos são as strings `flightDuration` e `flightCode`, o campo `stops` do tipo `i32`, as duas structs de `arrival` e `departure`, os vetores de `segments` e de `cabins`: 

```rust
#[derive(Serialize, Deserialize, Debug, PartialEq, Clone, GraphQLObject)]
#[serde(rename_all = "camelCase")]
pub struct Flight {
    flight_code: String,
    arrival: Location,
    departure: Location,
    stops: i32,
    segments: Vec<Segment>,
    flight_duration: String,
    cabins: Vec<Cabin>
}
```

Note que `arrival` e `departure` estão com o mesmo tipo, `Location`, pois possuem a mesma estrutura:

```json
{
    "airportCode":"POA",
    "airportName":"Salgado Filho",
    "cityCode":"POA",
    "cityName":"Porto Alegre",
    "countryCode":"BR",
    "date":"2020-07-21",
    "dateTime":"2020-07-21T09:15-03:00",
    "time":{ "stamp":"09:15", "hours":"09", "minutes":"15" }
}
```

Que se torna a struct `Location`, na qual quase todos os campos são Strings exceto por `time` que será a struct `Time`:

```rust
#[derive(Serialize, Deserialize, Debug, PartialEq, Clone, GraphQLObject)]
#[serde(rename_all = "camelCase")]
pub struct Location {
    airport_code: String,
    airport_name: String,
    city_code: String,
    city_name:String,
    country_code: String,
    date: String,
    date_time: String,
    time: Time,
}

#[derive(Serialize, Deserialize, Debug, PartialEq, Clone, GraphQLObject)]
pub  struct Time {
    stamp: String, 
    hours: String, 
    minutes: String, 
}
```

Para implementarmos `Segment` precisamos nos basear no seguinte json:

```json
{
    "flightCode":"LA4596",
    "flightNumber":"4596",
    "equipment":{ "name":"Airbus 321", "code":"321" },

    "airline":{
        "marketing":{ "code":"LA", "name":"LATAM Airlines" },
    },
    "duration":"PT1H35M",
    "departure": Location,
    "arrival": Location,
}
```

```rust
#[derive(Serialize, Deserialize, Debug, PartialEq, Clone, GraphQLObject)]
#[serde(rename_all = "camelCase")]
pub struct Segment {
    flight_code: String,
    flight_number: String,
    equipment: Info,
    airline: Airline,
    duration: String,
    departure: Location,
    arrival: Location,
}

#[derive(Serialize, Deserialize, Debug, PartialEq, Clone, GraphQLObject)]
pub struct Info {
    name: String,
    code: String,
}

#[derive(Serialize, Deserialize, Debug, PartialEq, Clone, GraphQLObject)]
pub struct Airline {
    marketing: Info,
}
```

Por último, podemos implementarmos `Cabin` e vamos nos basear no json:

```json
{
    "code":"Y",
    "displayPrice":101.03,
    "availabilityCount":18,
    "displayPrices":{
        "slice":101.03,
        "wholeTrip":101.03,
    },
    "fares":[...]
}
```

```rust
#[derive(Serialize, Deserialize, Debug, PartialEq, Clone, GraphQLObject)]
#[serde(rename_all = "camelCase")]
pub struct Cabin {
    code: String,
    display_price: f64,
    availability_count: i32,
    display_prices: DisplayPrice,
    fares: Vec<Fare>
}

#[derive(Serialize, Deserialize, Debug, PartialEq, Clone, GraphQLObject)]
#[serde(rename_all = "camelCase")]
pub struct DisplayPrice {
    slice: f64,
    whole_trip: f64,
}
```

Por último precisamos modelar `Fare` cujo Json é:

```json
{
    "code":"SL",
    "category":"LIGHT",
    "fareId":"250/BdC0XLNrp3H333zqIqmJJpNOO/05D8NwB5zcjHVNnkyl4GjqR/YOQrcDNWLPERAtEBLYqieN002",
    "availabilityCount":18,
    "price":{
        "adult":{
            "amountWithoutTax":68.9,
            "taxAndFees":32.13,
            "total":101.03
        }
    }
},
```

```rust
#[derive(Serialize, Deserialize, Debug, PartialEq, Clone, GraphQLObject)]
#[serde(rename_all = "camelCase")]
pub struct Fare {
    code: String,
    category: String,
    fare_id: String,
    availability_count: i32,
    price: Price
}

#[derive(Serialize, Deserialize, Debug, PartialEq, Clone, GraphQLObject)]
pub struct Price {
    adult: PriceInfo
}

#[derive(Serialize, Deserialize, Debug, PartialEq, Clone, GraphQLObject)]
#[serde(rename_all = "camelCase")]
pub struct PriceInfo {
    amount_without_tax: f64, 
    tax_and_fees: f64, 
    total: f64, 
}
```

Com a modelagem pronta, podemos finalizar a implementação da query `recommendations`.

## Implementando a Query `recommendations`

Como a estrutura de `recommendations` será praticamente igual a de `best_prices`, podemos nos antecipar e adicionar o resolver de `recommendations` na implementação de `QueryRoot`:

```rust
use crate::resolvers::internal::{best_prices_info, recommendations_info};
use crate::schema::{errors::{GenericError}, model::web::{best_prices::BestPrices, recommendations::Recommendations}};
// ...

#[juniper::object]
impl QueryRoot {
    // ...

    fn recommendations(
        departure: String,
        origin: String,
        destination: String,
    ) -> Result<Recommendations, GenericError> {
        error::iata_format(&origin, &destination)?;
        error::departure_date_format(&departure)?;
        let recommendations = recommendations_info(departure, origin, destination)?;
        Ok(recommendations)
    }
}
```

Vamos receber um erro indicando que `recommendations_info` não foi implementada ainda, e, assim, podemos partir para sua implementação, que seráexatamente igual a `best_prices_info`, apenas renomeando os campos de `best_prices` para `recommendations`:

```rust
use crate::boundaries::http_out::{best_prices, recommendations};
use crate::schema::{errors::GenericError, model::web::{best_prices::BestPrices, recommendations::Recommendations}};

/// ...

pub fn recommendations_info(
    departure: String,
    origin: String,
    destination: String,
) -> Result<Recommendations, GenericError> {
    let recommendations_text = recommendations(departure, origin, destination)?.text()?;
    let recommendations: Recommendations = serde_json::from_str(&recommendations_text)?;

    Ok(recommendations)
}
```

Agora precisamos implementar a função `http_out::recommendations` que também é parecida com a função `http_out::best_prices` pois possui apenas a `url` de request diferente:

```rust
// boundaries/http_out.rs
use reqwest::{blocking::Response, Result};
// ...

pub fn recommendations(departure: String, origin: String, destination: String) -> Result<Response> {
    let url =
        format!("https://bff.latam.com/ws/proxy/booking-webapp-bff/v1/public/revenue/recommendations/oneway?departure={}&origin={}&destination={}&cabin=Y&country=BR&language=PT&home=pt_br&adult=1&promoCode=",
                departure, origin, destination);
    reqwest::blocking::get(&url)
}
```

Agora podemos executar `cargo run` e brincar com a interface gráfica de nosso serviço em `localhost:4000/graphiql`. Um exemplo de request que podemos utilizar é:

```
{
  bestPrices(departure: "2020-07-21", 
    origin: "POA", 
    destination: "GRU") {
    bestPrices {
      date
      available
      price {amount}
    }
  }
  
  recommendations(departure: "2020-07-21", 
    origin: "POA", 
    destination: "GRU") {
    data {
      recommendedFlightCode 
      flights {
        flightCode
        flightDuration
        arrival {
          airportCode
          airportName
          dateTime
        }
        departure {
          airportCode
          airportName
          dateTime
        }
      }
    }
  }
}
```

* Lembre de atualizar as datas, quando este livro foi escrito elas estavam distantes.

A resposta parcial desta query é:

```json
{
  "data": {
    "bestPrices": {
      "bestPrices": [...]},
    "recommendations": {
      "data": [
        {
          "recommendedFlightCode": "LA4596",
          "flights": [
            {
              "flightCode": "LA4596",
              "flightDuration": "PT1H35M",
              "arrival": {
                "airportCode": "GRU",
                "airportName": "Guarulhos Intl.",
                "dateTime": "2020-07-21T10:50-03:00"
              },
              "departure": {
                "airportCode": "POA",
                "airportName": "Salgado Filho",
                "dateTime": "2020-07-21T09:15-03:00"
              }
            },
            {
              "flightCode": "LA4629",
              "flightDuration": "PT1H35M",
              "arrival": {
                "airportCode": "GRU",
                "airportName": "Guarulhos Intl.",
                "dateTime": "2020-07-21T22:20-03:00"
              },
              "departure": {
                "airportCode": "POA",
                "airportName": "Salgado Filho",
                "dateTime": "2020-07-21T20:45-03:00"
              }
            },
            {
              "flightCode": "LA3287",
              "flightDuration": "PT1H40M",
              "arrival": {
                "airportCode": "GRU",
                "airportName": "Guarulhos Intl.",
                "dateTime": "2020-07-21T16:55-03:00"
              },
              "departure": {
                "airportCode": "POA",
                "airportName": "Salgado Filho",
                "dateTime": "2020-07-21T15:15-03:00"
              }
            },
            {
              "flightCode": "LA3152LA3633",
              "flightDuration": "PT4H55M",
              "arrival": {
                "airportCode": "GRU",
                "airportName": "Guarulhos Intl.",
                "dateTime": "2020-07-21T13:40-03:00"
              },
              "departure": {
                "airportCode": "POA",
                "airportName": "Salgado Filho",
                "dateTime": "2020-07-21T08:45-03:00"
              }
            },
            {
              "flightCode": "LA3152LA3613",
              "flightDuration": "PT8H30M",
              "arrival": {
                "airportCode": "GRU",
                "airportName": "Guarulhos Intl.",
                "dateTime": "2020-07-21T17:15-03:00"
              },
              "departure": {
                "airportCode": "POA",
                "airportName": "Salgado Filho",
                "dateTime": "2020-07-21T08:45-03:00"
              }
            },
            {
              "flightCode": "LA3090LA3029LA3296",
              "flightDuration": "PT12H45M",
              "arrival": {
                "airportCode": "GRU",
                "airportName": "Guarulhos Intl.",
                "dateTime": "2020-07-21T22:10-03:00"
              },
              "departure": {
                "airportCode": "POA",
                "airportName": "Salgado Filho",
                "dateTime": "2020-07-21T09:25-03:00"
              }
            },
            {
              "flightCode": "LA3090LA3151LA3687",
              "flightDuration": "PT12H50M",
              "arrival": {
                "airportCode": "GRU",
                "airportName": "Guarulhos Intl.",
                "dateTime": "2020-07-21T22:15-03:00"
              },
              "departure": {
                "airportCode": "POA",
                "airportName": "Salgado Filho",
                "dateTime": "2020-07-21T09:25-03:00"
              }
            },
            {
              "flightCode": "LA3152LA3175LA3687",
              "flightDuration": "PT13H30M",
              "arrival": {
                "airportCode": "GRU",
                "airportName": "Guarulhos Intl.",
                "dateTime": "2020-07-21T22:15-03:00"
              },
              "departure": {
                "airportCode": "POA",
                "airportName": "Salgado Filho",
                "dateTime": "2020-07-21T08:45-03:00"
              }
            },
            {
              "flightCode": "LA3152LA3028LA3381",
              "flightDuration": "PT13H35M",
              "arrival": {
                "airportCode": "GRU",
                "airportName": "Guarulhos Intl.",
                "dateTime": "2020-07-21T22:20-03:00"
              },
              "departure": {
                "airportCode": "POA",
                "airportName": "Salgado Filho",
                "dateTime": "2020-07-21T08:45-03:00"
              }
            }
          ]
        }
      ]
    }
  }
}
```

Um exemplo utilizando `variables` pelo `Postman` pode ser a foto a seguir. A mesma estratégia poderá ser utilizada via `curl`.

![Usando `variables` com o Postman](imagens/postmanVariables.png)

Nosso próximo passo é implementar um sistema de caching com redis, para evitar múltiplos requests para a API da Latam.

[Anterior](02-bestprices.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](04-redis.md)