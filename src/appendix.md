[Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Apêndice](appendix.md) | [Bibliografia](bibliografia.md)

# Apêndice A - Benchmarks

## Techempower

Os benchmarks do Techempower são abertos a comunidade desenvolver seus serviços conforme acreditarem ser melhor e consistem em 5 categorias:
1. Serialização Json, neste teste cada resposta é uma instância com a chave `message` e o valor `Hello, World!` para ser serializada como Json. Actix em quinto lugar (rust em terceiro).
2. Única Query, neste teste cada request é processado fazendo fetch de uma única linha em uma simples tabela no bancl de dadps. A linha é então serializada em JSON. Actix em primeiro lugar.
3. Multiplas Queries, neste teste cada request processa multiplos fetchs em multiplas linhas de uma tabela simples e serializa essas linhas em Json. Actix em primeiro lugar.
4. Fortuna, neste teste, o ORM do framework é utilizado para fazer fetch de todas as linhas de uma tabela contendo um número desconhecido de mensagens de biscoitos da sorte. Um biscoito extra é adicionado na lista durante execução e após isso a lista é ordenada alfabeticamente. Actix em primeiro lugar.
5. Atualização de dados, neste teste verificamos a escrita em banco de dados. Cada request processa multiplos fetchs em multiplas linhas de uma tabela simples, guardamos em memória, modificamos os valores em memória, atualizamos cada linha individualmente  e depois serializamos os valores em memória para retornar como Json. Actix em primeiro lugar.
6. Texto puro, neste teste o framework responde com `plaintext` `"hello, World"`. Actix em quarto lugar (rust em primeiro e segundo).

![Techempower 2018 Fortunes](imagens/techempower.png)
Referência https://www.techempower.com/benchmarks/#section=data-r18&hw=ph&test=fortune

## Express vs Actix

Maxim Vorobjov criou seu próprio benchmark comparando comparando Express Node com Actix Rust e obteve resultados bastante interessantes. Ele comparou a performance, a estabilidade e o custo de um simples microserviço escrito com Express e com Actix. O setup estava limitado a 1 core. 

O cenário era um microserviço que permite os clientes procurarem por `task` que podem ou não estar associadas a `workers`. Assim, o banco de dados possui apenas duas tabelas, `WORKER` e `TASK`, na qual `TASK` possui um relação com `WORKER`:

```sql
CREATE TABLE worker (
	id SERIAL PRIMARY KEY,
	name varchar(255) NOT NULL,
	email varchar(255) NULL,
	score integer DEFAULT 0
);

CREATE TABLE task (
	id SERIAL PRIMARY KEY,
	summary varchar(255) NOT NULL,
	description text NOT NULL,
	assignee_id integer NULL REFERENCES worker,
	created TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### Teste de Carga

Teste de performance foi feito em um  Ubuntu 18 Xeon E5–2660 com 40 cores, e o banco de dados foi inicializado com 100_000 `TASK` aleatórias com 1000 `WORKER`s  relacionadas. Os testes foram executados com https://github.com/wg/wrk e os resultados são os seguintes:

![Requests por segundo, maior é melhor](imagens/actixexpressloadrps.png)

![Tempo médio de resposta, ms, menor é melhor](imagens/actixexpressloadmrt.png)

Continue lendo em https://medium.com/@maxsparr0w/performance-of-node-js-compared-to-actix-web-37f20810fb1a


# Apêndice B - Requests

## Todo Server

### Signup

`POST http://localhost:4000/auth/signup`

**Headers**
```json
{
    "Content-Type": "application/json",
    "x-customer-id": "d8f98c2e-07df-4ed6-8645-7f0b25536fdf"
}
```

**Body**
```json
{
	"email": "my@email.com",
	"password": "My cr4zy p@ssw0rd My cr4zy p@ssw0rd"
}
```

### Login

`POST http://localhost:4000/auth/login`

**Headers**
```json
{
    "Content-Type": "application/json",
    "x-customer-id": "d8f98c2e-07df-4ed6-8645-7f0b25536fdf"
}
```

**Body**
```json
{
	"email": "my@email.com",
	"password": "My cr4zy p@ssw0rd My cr4zy p@ssw0rd"
}
```

### Logout

`DELETE http://localhost:4000/auth/logout`

**Headers**
```json
{
    "Content-Type": "application/json",
    "x-customer-id": "d8f98c2e-07df-4ed6-8645-7f0b25536fdf",
    "x-auth": "eyJhbGciOiJIUzI1NiIsImRhdGUiOiIyMDIwLTAyLTI4IDAxOjQxOjU2LjA2NjYxNTQwMCBVVEMiLCJ0eXAiOiJqd3QifQ.eyJlbWFpbCI6Im15QGVtYWlsLmNvbSIsImV4cGlyZXNfYXQiOiIyMDIwLTAyLTI5VDAxOjQxOjU2LjA2MzI2ODgwMCIsImlkIjoiZDdjNTk1MTItYjlhYS00NzBhLWEwNjUtZTAwYTYxMTcxYmE0In0.gIycarcQhbbcjvYIHDW_9fVgCFrFs1LjlJFMZGIm_kw"
}
```

**Body**
```json
{
	"email": "my@email.com"
}
```

### Create Todo

`POST http://localhost:4000/api/create`

**Headers**
```json
{
    "Content-Type": "application/json",
    "x-customer-id": "d8f98c2e-07df-4ed6-8645-7f0b25536fdf",
    "x-auth": "eyJhbGciOiJIUzI1NiIsImRhdGUiOiIyMDIwLTAyLTI4IDAxOjQxOjU2LjA2NjYxNTQwMCBVVEMiLCJ0eXAiOiJqd3QifQ.eyJlbWFpbCI6Im15QGVtYWlsLmNvbSIsImV4cGlyZXNfYXQiOiIyMDIwLTAyLTI5VDAxOjQxOjU2LjA2MzI2ODgwMCIsImlkIjoiZDdjNTk1MTItYjlhYS00NzBhLWEwNjUtZTAwYTYxMTcxYmE0In0.gIycarcQhbbcjvYIHDW_9fVgCFrFs1LjlJFMZGIm_kw"
}
```

**Body**
```json
{
	{
	"title": "title",
	"description": "descrition",
	"state": "Done",
	"owner": "90e700b0-2b9b-4c74-9285-f5fc94764995",
	"tasks": [
		{
			"is_done": true,
			"title": "blob"
			
		},
		{
			"is_done": false,
			"title": "blob2"
			
		}]
}
}
```

### Get All Todos

`GET http://localhost:4000/api/index`

**Headers**
```json
{
    "Content-Type": "application/json",
    "x-customer-id": "d8f98c2e-07df-4ed6-8645-7f0b25536fdf",
    "x-auth": "eyJhbGciOiJIUzI1NiIsImRhdGUiOiIyMDIwLTAyLTI4IDAxOjQxOjU2LjA2NjYxNTQwMCBVVEMiLCJ0eXAiOiJqd3QifQ.eyJlbWFpbCI6Im15QGVtYWlsLmNvbSIsImV4cGlyZXNfYXQiOiIyMDIwLTAyLTI5VDAxOjQxOjU2LjA2MzI2ODgwMCIsImlkIjoiZDdjNTk1MTItYjlhYS00NzBhLWEwNjUtZTAwYTYxMTcxYmE0In0.gIycarcQhbbcjvYIHDW_9fVgCFrFs1LjlJFMZGIm_kw"
}
```

## Recomendations GraphQL

### GraphQL:
-  Graphich interface: http://localhost:4000/graphiql
-  Para requests: 

### API Externa

- Recomendações de voos: https://bff.latam.com/ws/proxy/booking-webapp-bff/v1/public/revenue/recommendations/oneway?departure=<YYYY-mm-dd>&origin=<IATA>&destination=<IATA>&cabin=Y&country=BR&language=PT&home=pt_br&adult=1&promoCode=

- Melhores preços https://bff.latam.com/ws/proxy/booking-webapp-bff/v1/public/revenue/bestprices/oneway?departure=<YYYY-mm-dd>&origin=<IATA>&destination=<IATA>&cabin=Y&country=BR&language=PT&home=pt_br&adult=1&promoCode=

[Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Apêndice](appendix.md) | [Bibliografia](bibliografia.md)