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
        ...
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

    //...
}
```

Agora podemos criar o módulo `recommendations` com:

```rust
pub mod recommendations {
    use juniper::GraphQLObject;
    use serde::{Deserialize, Serialize};
    //...
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