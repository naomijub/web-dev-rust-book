[Anterior](../intro/5-exercício.md) | [Topo](https://github.com/naomijub/web-dev-rust-book/blob/master/book.md) | [Próximo](./01-ping-pong.md)

# Todo Server com Actix 

Nesta primeira parte vamos desenvolver um Todo Server aplicando um modelo semelhante ao MVC do projeto Phoenix, da comunidade Elixir. Este modelo foi explicado no capítulo `Como este livro é organizado`, mas agora vamos detalhar um pouco quais serão as características deste serviço.

Vamos criar um RESTful Todo Server que seria facilmente utilizado em produção pois contará com uma série de recursos como:
1. Endpoints de monitoramento:
    - `ping` que funciona como `health check`.
    - `~/ready` que funciona como disponibilidade do servico, `readiness probe`.
2. Endpoints para salvar as informações dos `TODO`s, `create` `show` `show-by-id` e `update`.
3. Sistema de logs, headers padrão e middlewares de autenticação.
4. Endpoints de autenticação, com `signup`, `login` e `logout` utilizando tokens `JWT` e banco de dados Postgres via Diesel.
5. Bastion para tornar o sistema tolerante a falhas.
6. Dockerização de todos os serviços.
7. CI executando as pipelines de teste.
8. Serde para serialização e deserialização de Json.