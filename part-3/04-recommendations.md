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

Adicionamos esta query a função `fetch_gql` que executará ambas as queries simultaneamente. E já modificamos a struct `GqlField` para conter o campo `recommendations`:

```rust
pub fn fetch_gql() -> Value {
    json!({
        "query": "{
            recommendations(departure: \"2020-06-28\", 
                origin: \"POA\", 
                destination: \"GRU\") {
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
            bestPrices(departure: \"2020-06-28\", origin: \"POA\", destination: \"GRU\") {
                bestPrices {
                    date
                    available
                    price {amount}
                }
             }
        }"
    })
}


#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
#[serde(rename_all = "camelCase")]
pub struct GqlFields {
    best_prices: BestPrices,
    recommendations: Recommendations,
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

Para este componente vamos utilizar uma forma diferente de criação de `view`. A ideia aqui é demonstrar como ficaria a organização de subcomponentes que possuem seu prórpio estado e depois mostrar como gerenciar estas informações. Agora, na implementação da struct `GqlResponse` precisamos adicionar uma função que facilite acesso ao campo privado `recommendations`, fazemos isso com:

```rust
impl GqlResponse {
    pub fn best_prices(self) -> BestPrices {
        self.data.best_prices
    }

    pub fn recommendations(self) -> Recommendations {
        self.data.recommendations
    }
}
```

Com isso, podemos chamar a mesma estratégia que usamos com `best_prices` e chamar a função `view` dentro da `view` de `Airline`. Para podemos chamar doiis componentes `Html` dentro de outro, vamos preciisar separar em várias `div`s, pois cada conjunto de tag permiite a utilização de apenas um `Html` do `yew`. fazemos isso modificandoa  função `view` de `app.rs`:

```rust
fn view(&self) -> Html {
    if self.fetching {
        html! {
            <div class="loading-margin">
                <div class="loader"></div>
            </div>
        } 
    } else {
        html! {
            <div class="body">
                { 
                    if let Some(data) = &self.graphql_response {
                        html!{<div>
                            <div> {data.clone().best_prices().view()} </div>
                            <div> { data.clone().recommendations().view() } </div>
                          </div> }
                    } else {
                        html!{
                            <p class="failed-fetch">
                                {"Failed to fetch"}
                            </p>
                        }
                    }
                }
            </div>
        }
    }
}
```

Agora precisamos definir a função ` view` para a struct `Recommendations`, faremos isso utilizando a trait `Component`, que por enquanto não possuirá nenhuma mensagem, `Msg`, mas que receberá `Recommendations` como `Property`:

```rust

pub enum Msg {
}

impl Component for Recommendations {
    type Message = Msg;
    type Properties = Recommendations;

    fn create(rec: Self::Properties, link: ComponentLink<Self>) -> Self {
        rec
    }

    fn change(&mut self, _: <Self as yew::html::Component>::Properties) -> bool { 
        false
     }

    fn update(&mut self, _msg: Self::Message) -> ShouldRender {  
        true
    }

    fn view(&self) -> Html {
      html!{
          <div class="flight-container"> {
              self.data[0].clone().flights.into_iter()
              .map(|r|
                  html!{
                      <div class="flight"> 
                          // ...
                      </div>
                  }
              )
              .collect::<Html>()
          }
          </div>}
    }
}
```

Para que `Recommendations` possa ser do tipo `Properties`, precisamos implementar a macro derive `Properties` em cada uma das structs de `Recommendations`. Fazemos isso adicionando ela ao `derive` de todas elas:

```rust
#[derive(Properties, Serialize, Deserialize, Debug, PartialEq, Clone)]
pub struct Recommendations {
    data: Vec<RecommendedFlight>
}
```

Uma vez que essa macro esteja implementada podemos garantir que ao serializarmos a struct `Recommendations`, seu valor vai ser passado como argumento para a função `create` que vai nos permiitir retornar o estado inicial do `Component` com os valores correstos:

```rust
fn create(rec: Self::Properties, link: ComponentLink<Self>) -> Self {
    rec
}
```

### Compondo a `view`

Anteriormente descrevemos a função `view` da seguinte forma:

```rust
fn view(&self) -> Html {
  html!{
      <div class="flight-container"> {
          self.data[0].clone().flights.into_iter()
          .map(|r|
              html!{
                  <div class="flight"> 
                      // ...
                  </div>
              }
          )
          .collect::<Html>()
      }
      </div>}
}
```

A primeira coisa que gostaria de salientar é a presenção do `clone` logo após acessarmos o vetor com `[0]`, isso se deve ao fato de que temos apenas a referência de um valor, que não vai nos possibilitar executar o `move` para a macro, conforme o erro a seguir:

```
  --> src/reccomendation.rs:74:17
   |
74 |                 self.data[0].flights.into_iter()
   |                 ^^^^^^^^^^^^^^^^^^^^ move occurs because value has type `std::vec::Vec<reccomendation::Flight>`, which does not implement the `Copy` trait
   |
```

A criação do `flight-container` é algo bastante simples, pois basta incluirmos a `div` dentro da macro `html!`, mas dentro do container precisamos iterar sobre cada um dos voos utilizando `self.data[0].clone().flights.into_iter()`. Note que o campo `data` poderia ser `Option` e neste caso deveriamos utilizar um `if let Some(flights)` para extrair valor de `Some(flights)`, ou poderia ser um vetor vazio, e neste caso seriia melhor utilizar a função `first` e aplicar um `match`. Uma vez que temos nosso `into_iter` podemos applicar um `map` que vai criar nossos `Htmls`, para depois colecionarmos em um `collect::<Html>()`. Nosso `map` começa com um `html!` que cria a div correspondete a todo o bloco com informações do voo, conforme a imagem a seguir:

![Componente com cada uma das recomendações de voo](../imagens/flights.png)

O css para está parte é este:

```css

.flight-container {
  width: 70%;
  margin-top: 5rem;
  transform: translate(35%, 0%);
  background-color: #f1f0f0;
}

.flight {
  display: table-row;
  border: 1px solid lightgrey;
}

.flight-cell {
  padding: 2rem 1rem;
  display: table-cell;
  border-bottom: 1px solid lightgray;
}

.origins-destinations {
  width: 22rem;
  display: flex;
}

.arrow {
  font-size: 30px;
  font-weight: bold;
  color: red;
  margin-left: 2rem;
  margin-right: 2rem;
}

.origin-destination {
  font-size: 26px;
  display: flex;
}

.time {
  font-weight: bold;
  color: black;
}

.airport {
  color: rgb(75, 74, 74);
  margin-left: 1rem;
}

.duration {
  padding: 1rem 2rem;
  font-size: 20px;
  color: gray;
}

.stops {
  padding: 1rem 2rem;
  font-size: 20px;
  color: blue;
}

.price {
  font-size: 26px;
  font-weight: bold;
  padding: 1rem 2rem;
  margin-left: 1rem;
  border-left: 1px groove lightgray;
}
```

Assim, podemos começar a implementar cada uma das células que formam um voo. Fazemos isso utilizando várias `div`s com suas configurações definidas no CSS:

```rust
<div class="flight"> 
    <div class="flight-cell origins-destinations">
        <div class="origin-destination"> 
            <div class="time"> {{
                let date = Utc.datetime_from_str(&r.departure.date_time[..16],"%Y-%m-%dT%H:%M");
                date.unwrap().format("%H:%M").to_string()
            }} </div>
            <div class="airport"> {r.departure.airport_code} </div>
        </div>
        <div class="arrow">{">"}</div>
        <div class="origin-destination">
            <div class="time"> {{
                let date = Utc.datetime_from_str(&r.arrival.date_time[..16],"%Y-%m-%dT%H:%M");
                date.unwrap().format("%H:%M").to_string()
            }} </div>
            <div class="airport"> {r.arrival.airport_code} </div>
        </div>
    </div>
    <div class="flight-cell duration"> {
        r.flight_duration.replace("PT","").replace("H", "h ").replace("M", "min")
    } </div>
    <div class="flight-cell stops"> {
        if r.stops == 0 {
            html!{<p>{"Direto"}</p>}
        } else {
            html!{<p>{r.stops.to_string()}</p>}
        }   
    } </div>
    <div class="flight-cell price"> {"R$ 582,03"}</div>
</div>
```

A primeira célula, `origins-destinations`, possui três grandes campos, origem, seta e destino. Origem e destino são iguais em organizacão, mudando apenas o campo para acessar as iinformações de oirgem, `departuure`, e de destino, `arrival`. Existe uma pegadiinha no tiipo `date_time`, que torna parsear ele um pouco complicado para um  parser normal, pois ele não apresenta o campo de  segundos no horário, mas apresenta o timezone `-3:00`. A forma como podemos lidar com isso é enviando um formater que vai parsear apenas a parte da String de `dataTime` que temos controle, `"%Y-%m-%dT%H:%M"`, ou seja, os 16 primeiros caracteres, por isso do slice `r.departure.date_time[..16]`.

Nos campos faltantes, vamos aplicar algumas funções para alterar seu valor. Por exemplo, o campo `flight_duration` aparece no formato `PT1H40M`, mas queremos exibir o formato `1h 40min`, por isso removemos `PT`, fazemos o replace de `H` por `h ` e modificamos `M` para `min`. Quanto ao campo `stops`, a úncia coisa que precisamos garantir é que se a quantidade for zero, vamos escrever `Direto`, caso não for, mostramos quantas escalas são. O preço veremos a seguir.

## Introduzindo cabines