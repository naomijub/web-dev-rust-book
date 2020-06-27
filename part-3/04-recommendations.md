# Componente de Recomendações

A query que vamos utilizar neste capítulo será a de `recommendations`, que contém muitas mais informações que a de `best_prices`, mas será útil para compor os componentes faltantes. A baixo a query de `recommendations`:

```graphql
{
recommendations(departure: "2020-06-28", 
    origin: "POA", 
    destination: "GRU") {
    data{
      recommendedFlightCode
      flights {
        flightCode
        flightDuration
        stops
        arrival {
          cityName
          airportName
          airportCode
          dateTime
        }
        departure {
          cityName
          airportName
          airportCode
          dateTime
        }
        segments {
          flightNumber
          equipment {
            name
            code
          }
        }
        cabins {
          code
          displayPrice
          availabilityCount
        }
      }
    }
  }
}
```

Esta query retorna o seguinte Json:

```json
{
  "data": {
    "recommendations": {
      "data": [
        {
          "recommendedFlightCode": "LA3068",
          "flights": [
            {
              "flightCode": "LA3068",
              "flightDuration": "PT1H35M",
              "stops": 0,
              "arrival": {
                "cityName": "São Paulo",
                "airportName": "Guarulhos Intl.",
                "airportCode": "GRU",
                "dateTime": "2020-06-28T22:15-03:00"
              },
              "departure": {
                "cityName": "Porto Alegre",
                "airportName": "Salgado Filho",
                "airportCode": "POA",
                "dateTime": "2020-06-28T20:40-03:00"
              },
              "segments": [
                {
                  "flightNumber": "3068",
                  "equipment": {
                    "name": "Airbus 320-200",
                    "code": "320"
                  }
                }
              ],
              "cabins": [
                {
                  "code": "Y",
                  "displayPrice": 582.03,
                  "availabilityCount": 33
                },
                {
                  "code": "W",
                  "displayPrice": 1072.03,
                  "availabilityCount": 9
                }
              ]
            },
            {
              "flightCode": "LA3297",
              "flightDuration": "PT1H40M",
              "stops": 0,
              "arrival": {
                "cityName": "São Paulo",
                "airportName": "Guarulhos Intl.",
                "airportCode": "GRU",
                "dateTime": "2020-06-28T10:45-03:00"
              },
              "departure": {
                "cityName": "Porto Alegre",
                "airportName": "Salgado Filho",
                "airportCode": "POA",
                "dateTime": "2020-06-28T09:05-03:00"
              },
              "segments": [
                {
                  "flightNumber": "3297",
                  "equipment": {
                    "name": "Airbus 320-200",
                    "code": "320"
                  }
                }
              ],
              "cabins": [
                {
                  "code": "Y",
                  "displayPrice": 842.03,
                  "availabilityCount": 1
                },
                {
                  "code": "W",
                  "displayPrice": 1137.03,
                  "availabilityCount": 4
                }
              ]
            }
          ]
        }
      ]
    }
  }
}
```

Ou seja, nosso backend retornará um campo `data`, que pode ser um array vazio, pois pode retornar sem voos. O campo data por sua vez, retornará uma estrutura de `RecommendedFLights` que é um vetor com o voo recomendado, `recommendedFlightCode`, e todos os possíveis voos em `flights`. No primeiro momento, vamos focar apenas a estrutura do seguinte Json:

```json
{
    "flightCode": "LA3068",
    "flightDuration": "PT1H35M",
    "stops": 0,
    "arrival": {
        "cityName": "São Paulo",
        "airportName": "Guarulhos Intl.",
        "airportCode": "GRU",
        "dateTime": "2020-06-28T22:15-03:00"
    },
    "departure": {
        "cityName": "Porto Alegre",
        "airportName": "Salgado Filho",
        "airportCode": "POA",
        "dateTime": "2020-06-28T20:40-03:00"
    },
}
```

## Modelando Recommendations

A  modelagem do domínio vai seguir o mesmo formato do capítulo anterior e da parte anterior, assim podemos resumir a modelagem de `Recommendations` inicialmente da seguinte forma:

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
pub struct Recommendations {
    data: Vec<RecommendedFlight>
}

#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
#[serde(rename_all = "camelCase")]
pub struct RecommendedFlight {
    recommended_flight_code: String,
    flights: Vec<Flight>
}

#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
#[serde(rename_all = "camelCase")]
pub struct Flight {
    flight_code: String,
    flight_duration: String,
    stops: i32,
    arrival: OriginDestination,
    departure: OriginDestination,
    segments: Vec<Segment>
}

#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
#[serde(rename_all = "camelCase")]
pub struct OriginDestination {
    city_name: String,
    airport_name: String,
    airport_code: String,
    date_time: String,
}

#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
#[serde(rename_all = "camelCase")]
pub struct Segment { 
    flight_number: String,
    equipment: Equipment
}

#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
pub struct Equipment { 
    name: String,
    code: String
}
```

Agora podemos começar a implementar o componente da struct `Recommendations`.

## Implementando a `view` para `Recommendations`