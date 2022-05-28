# Instalando Rust

O primeiro passo para instalar Rust é a instalação do `rustup`, uma ferramenta de linha de comando para gerenciar versões do Rust e todo ferramental a sua volta. Para fazer download do `rustup` em Linux e macOS, basta digitar `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh` em seu terminal e seguir as instruções. Para a instalação em Windows, basta ir ao site https://www.rust-lang.org/tools/install e garantir que você possui o `C++ build tools for Visual Studio 2013` ou posterior instalado em sua máquina. 

Para atualizar o `rustup`, basta digitar `rustup update` em seu terminal e, para desinstalar o Rust, `rustup self uninstall`. Caso você precise saber a versão do Rust, basta digitar `rustc --version` e você verá a resposta no formato `rustc x.y.z (abcabcabc yyyy-mm-dd)`, na qual `x.y.z` correspondente à versão, `abcabcabc` ao commit da versão `x.y.z` e `yyyy-mm-dd` à data da versão `x.y.z`. Se estiver desconectada da internet e quiser ver a documentação, basta digitar `rustup doc`.

Não esqueça de experimentar a utilização do `cargo`, gerenciador de pacotes e de build do Rust. Para ver se ele está bem em sua máquina, basta digitar `cargo --version`. Para criar um pacote, você pode digitar `cargo new <nome do pacote> --lib` e, para criar um executável, você pode digitar `cargo new <nome do executável> --bin`. Caso você omita as opções `--lib` e `--bin`, o padrão atual é criar um executável. 

A minha experiência de desenvolvimento Rust tem sido muito agradável com o Racer e o RLS configurados no VSCode ou no emacs. Para o VSCode, basta adicionar os plugins `Rust` e `Rust (rls)`. Caso seu path do cargo tenha algum problema, será necessário apontar o caminho para o `racer` dentro do pacote do `cargo`.

[Update]: https://rust-analyzer.github.io/

Pronto, agora podemos resolver um exercício básico.
