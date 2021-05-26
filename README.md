# Desenvolvimento Web com Rust

**Livro sobre desenvolvimento web em Rust**
Autora: Julia Naomi Boeira

## Organização

Este livro está organizado da seguinte forma. Nos capítulos de introdução teremos um guia de instalação e um exercício básico. Nosso primeiro projeto será um servidor de gerenciamento de tarefas, baseado em Actix-web. Actix-web é um framework de desenvolvimento web em Rust que tem feito muito barulho pelas suas constantes primeiras colocações nos benchmarks de performance, principalmente da Techempower, e algumas confusões pelo uso de `unsafe`. Acredito que o framework seja bastante simples de entender e muito completo, mais informações sobre os benchmarks no apêndice A. Vamos utilizar diversas ferramentas da stack Actix para garantir um serviço completo e algumas ferramentas clássicas da comunidade Rust como Serde para serialização de Json e clientes de bancos de dados. Para este projeto vamos utilizar DynamoDB como banco de dados via biblioteca Rusoto para salvar os dados de gerenciamento de tarefas e Postgres com Diesel para salvar informações de autorização. O padrão de arquitetura de software utilizando para está aprte é inspirado no framework Phoenix do Elixir. 

Na segunda parte vamos modelar um serviço que retorna valores e rotas de passagens aéreas via queries GraphQL. Esse serviço realizará requests pare serviços reais e será desenvolvido no padrão de arquitetura Hexagonal, ou também conhecido como Cebola, que é inspirado no livro **Clean Architecture** de Robert C. Martin. A idéia central deste padrão é separarmos funções puras na parte mais interna da arquitetura e funções impuras na parte mais externa. A terceira parte do livro, que está associada a parte anterior, será a construção de um frontend utilizando a Yew Stack, que é baseada em WebAssembly, para a representação das opções de passagens. Lembrando que Html e Css serão mantidos de forma primitiva por não serem o foco deste livro.

* Para a parte 2 do livro, recomendo conhecer os conceitos de GraphQL, livros e tutoriais estão indicados na bibliografia. 
* Para a parte 3 do livro, recomendo conhecer os conceitos de WebAssembly, livro oficial gratuíto indicado na bibliografia.

### Por que do DynamoDB 
 
Existem dois motivos para eu ter escolhido utilizar o DynamoDB para nosso serviço.
1. Existem centenas de ótimos exemplos utilizando o banco Postgres com a crate Diesel. Inclusive integrando com AWS e em Português. Além do mais, vamos utilizar o Postgres com Diesel, só que não como nosso principal banco de dados, já que é bastante comum aplicações diferentes possuirem mais de um tipo de banco de dados. A escolha do Diesel para o middleware de autorização não tem nenhuma relação com segurança ou performance, somente foi o recurso que me ocorreu usar no momento.
2. Em um mundo cada vez mais voltado ao cloud, escolher uma tecnologia nativa de cloud parece uma boa solução.
Agora, isso não quer dizer que o DynamoDB seja o banco que modela perfeitamente nosso domínio ou as relações entre ele, mas é um banco com uma performance excepcional que permite muita flexibilidade ao modelar domínios. Alguns limites associados a transações e typos no DynamoDB:
* O tamanho máximo de uma String é limitada a `400KB`, assim como para binários. 
* Uma String de expressão pode ter no máximo `4KB`.
* Uma transação não pode conter mais de 25 itens únicos, assim como não pode conter mais de `25MB` de dados.
* É possível ter até 50 request simultâneos para criar, atualizar ou deletar tabelas.
* `BatchGetItem`, buscar um conjunto de itens pode trazer no máximo 100 itens e um total de `16MB` de dados.
* `BatchWriteItem`, como `PutItem` e `DeleteItem` podem conter até 25 itens e um total de `16MB` de dados.
* `Query` e `Scan` tem um limite de `1MB` por chamada.

### Phoenix MVC
 
O que eu gosto no modelo que o Phoenix utiliza para organizar seus módulos é a divisão entre a lógica web e a lógica core, ou seja, separa a camada de comunicação com o mundo da camada de comunicação interna. Por exemplo, uma API chamada de `TodoApi` vai possuir módulos com os nomes `todo_api_web` para a lógica web e `todo_api` para a lógica core. O formato interno do `todo_api_web` é bastante comum e geralmente utiliza a nomenclatura do `MVC`, na qual seus modelos representando o domínio estão dentro de um módulo chamado `models`, seus operadores entre camadas, geralmente sem lógica, ou controllers, estão em um módulo chamado `controllers`, e as visualizações das telas estão em um módulo chamado `views`. Caso você esteja utilizando `GraphQL` a divisão do MVC pode ficar um pouco diferente, como `queries` e `mutations` em um módulo de `schemas`, seus objetos de entrada e saída em um módulo chamado `models` e seus resolvers em um módulo resolvers, seria esse padrão o `MRS` (*Models Resolvers Schema*)?

Quanto ao módulo `todo_api`, ele estaria organizado com um módulo para gerenciar a fonte dos dados, geralmente denominado `repo` ou `db`, e um outro módulo para organizar as estruturas de dados correspondentes chamado de `models`. Aqui é comum existir uma camada que adapta dos modelos de `todo_api` para `todo_api_web`, geralmente chamado de `adapters`. Para serviços que se comunicam por mensagens, é comum um módulo `message` aqui também. Resumindo graficamente seria:

```
todo_api
    main
    |-> todo_api
        |-> adapters
        |-> db (ou repo)
        |-> message
        |-> models
    |-> todo_api_web
        |-> controllers
        |-> http (configurações do sistema e middlewares)
        |-> models
        |-> routes (rotas do sistema)
        |-> views
```

### Hexagonal

O modelo hexagonal é mais simples em organização, e talvez mais fácil para quem estiver começando a trabalhar em um sistema, mas talvez menos prático para quem já conhece o sistema. Ele consiste em algumas camadas que vão das camadas impuras com efeitos colaterais chamadas de `boundaries`. No módulo `boundaries` vamos ter coisas como `web`, `db` e `messaging`, ou seja, qualquer coisa que cause efeitos colaterais. Depois disso vamos ter uma camada que recebe esses efeitos colaterais e chama funções puras para lidar com eles, comumente chamado de `controllers`. Os `controllers` utilizam principalmente duas camadas para tratar os efeitos colaterais, a camada de `adapters`, que transforma as entidades de `boundaries` em entidades internas, e a camada de `core`, que é dona de toda lógica do sistema, `core` pode ainda possuir uma separação de negócio, `business`, e outra computacional ou de apoio, `compute`. Por último, a parte mais interna são os `models`, que correspondem a estrutura de dados do sistema. Resumindo graficamente, em ordem de mais impura até mais pura, seria:

```
todo_api
    main
    |-> boundaries
        |-> web
        |-> db
        |-> message
    |-> controllers/resolvers
    |-> adapters
    |-> core 
        |-> business
        |-> compute
    |-> models/schemas
```

* Qual nomenclatura ou quais nomes específicos você vai dar para seus módulos é menos relevante do que a forma como as coisas estarão organizadas, a única coisa importante de lembrar é a necessidade de seu código e sua organização ser inteligivel para todas as pessoas.

* Este livro utiliza apenas o framework Actix para servidores web, mas exemplos com outros frameworks podem ser encontrados no livro `Programação Funcional e Concorrente em Rust`. Actix é o framework de `actors` do Rust, e tem como framework web o `actix-web`. 

* A versão de Rust utilizada neste livro é a 1.40 da edição 2018, assim, caso a linguagem evolua mais rápido que o livro, você pode fazer Pull Request nos repositórios do livro. Só peço que explique no Pull Request a que parte esse Pull Request se refere, qual a modificação, o porquê da modificação e caso ela derive de um erro pré existente, salientar o motivo. Algumas modificações de organização e renomear arquivos podem ser bastante interessantes também.

## Sobre a autora

Julia Naomi Boeira é uma engenheira de software com experiência em programação funcional e concorrente, games e sistemas distribuídos. Atualmente trabalha com open source em Rust e C++. Algumas empresas em seu currículo são Ubisoft, Creditas, Chorus One, Nubank, Thoughtworks, Latam e Globo.com.

Para este livro tenho a agradecer o Otávio Pace, o Bruno Tavares e o pessoal da Rust in Poa (Julio Biason, Douglas e Ruan Nunes, e todos os exercícios do Exercism.io que fizemos).
