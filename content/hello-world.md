+++
title = "println!(\"Hello World!\")"
description = "Blog, como nasceu, onde está, como vive. Hoje no Rust Brasil"
date=2018-05-28
category = "Apresentação"
tags = ["amigavel a iniciantes", "introdução", "gutenberg"]

[extra]
author="Guilherme Diego"
+++

Olá, me chamo Guilherme Diego e sou muitas coisas nessa vida e uma delas é ser entusiasta de coisas que em meu ponto de vista são boas, e esse foi o meu caso com RUST. Desde que comecei minha jornada pelo maravilhoso mundo de rust, eu tenho recebido ajuda da incrível comunidade que é a Rust Brasil pelo [telegram](), e já não é recente o assunto de: *"Devíamos criar um blog"*. Eu sou *front-end* já fazem quase 6 anos, e me responsabilizei por erguer os templates e tomar a frente de organizar o blog junto com outras pessoas que queriam dividir conhecimentos. O que nos faz chegar ao motivo dessa postagem: Introduzir como o Blog foi feito e como contribuir para ele.

## Gutenberg (ou Dinkleberg)
Quando iniciamos nossas discussões para o que usaríamos para erguer o blog estático no Github, e optamos pelo [**Gutenberg**](), uma alternativa a grandes nomes como Hugo, Jenkyll e Cobalt, porem, escrito em Rust <3. Gutenberg tem uma mecânica muito simples, ele é composto por 4 pastas:

- **content**: Onde fica as páginas
- **templates**: Onde fica os *templates* a serem rendenizados
- **sass**: Onde fica os estilos em `scss`
- **themes** Onde fica os temas

Por incrível que pareça, um tema nada mais é que um site Gutenberg sobrescrito por informações de outro site Gutenberg! Bruxaria? Talvez! Logo foi dado início a um tema do zero para o nosso Blog. Logo a primeira vez que ouvi o nome da ferramenta, eu um amante de padrinhos mágicos, já cerrei meus olhos e disse lentamente: "Dinkleberg", acabamos aderindo em `alias`, até que eu batizei o tema que escrevemos como:

![753986_1](https://user-images.githubusercontent.com/10289071/40806112-dd79ae78-64f6-11e8-8f24-63f387d5bb8f.jpg)

Logo, caso sinta vontade de contribuir com o tema você pode achar ele [aqui](). Se você deseja publicar algo irei agora explicar como publicar.

## Contribuindo
Existem dois jeitos de contribuir: MANUALMENTE pelo seu computador, ou direto pelo Github. Irei mostrar ambas!

### Manualmente
- Faça um fork do repositório do blog: https://github.com/rust-br/blog
- Clone ele na sua maquina
- Entre na pasta
- `git submodule init`
- `git submodule update`
- Crie um arquivo na raiz do `/content` com o `slug` do seu artigo, exemplo: `borrow-checker-para-iniciantes.md`
- Escreva / commit / push
- Abra um pull request para o master go rust-br/blog

OBS: Caso queira rodar local, instale o [gutenberg]() e rode `gutenberg --config config.toml serve`

### Pelo Github
- Faça um fork do repositório do blog: https://github.com/rust-br/blog
- Crie um branch
  - <img width="282" alt="captura de tela 2018-05-31 as 17 31 54" src="https://user-images.githubusercontent.com/10289071/40806737-c43910be-64f8-11e8-9070-487d052ae4dc.png">
- Pelo próprio github acesse a pasta content, e clique em (Create new file / Criar novo arquivo no canto superior direito)
  - <img width="898" alt="captura de tela 2018-05-31 as 17 32 19" src="https://user-images.githubusercontent.com/10289071/40806738-c462e2cc-64f8-11e8-96f7-c4764ab1fd7d.png">
- Escreva seu post e commit
  - <img width="917" alt="captura de tela 2018-05-31 as 17 32 39" src="https://user-images.githubusercontent.com/10289071/40806739-c483539a-64f8-11e8-83af-42f28b217d59.png">
- abra um pull request para o master go rust-br/blog

OBS: É possível editar os arquivos pelo github! Uma vez que eles estejam na pasta `/content`, basta clicar no arquivo que deseja editar e posteriormente no lápis no canto superior direito
<img width="903" alt="captura de tela 2018-05-31 as 17 37 05" src="https://user-images.githubusercontent.com/10289071/40806989-76cbc4f6-64f9-11e8-8e5d-5d3bccdaa1d5.png">

## Boas Práticas
É necessário que se siga boas práticas para que o post tenha o resultado necessário, algumas informações são obrigatórias quando criar o post!

### Cabeçalho

```markdown
+++
title = "println!(\"Hello World!\")" # TITULO DO POST
description = "Blog, como nasceu, onde está, como vive. Hoje no Rust Brasil" # DESCRIÇÃO DO POST
date=2018-05-28 # DATA DE PUBLICAÇÃO
category = "Apresentação" # Categoria da publicação
tags = ["amigável a iniciantes", "introdução", "gutenberg"] # Tags relacionadas

[extra]
author="Guilherme Diego" # Nome do autor (mantenha o padrão entre suas postagens :D)
updated_date=2018-05-29 # Data que você editou o post
+++
```

O **updated_date** é o único parâmetro NÃO OBRIGATÓRIO do header! Posts sem os requisitos serão negados. Você pode pedir ajuda a qualquer momento para encaixar seu post em uma **Categoria** e colocar as **Tags** nela, estaremos sempre dispostos a ajudar.

### Branch
É recomendado que use a nomenclatura `post/nome-do-meu-post` quando for inserir um novo post no blog e `repost/nome-do-meu-post` quando for atualizar um post. Em caso de atualização de tema ou ajuste em templates ou outras coisas use a nomenclatura `fix/resumo-do-que-fez`

### Português Por Favor
Somos um blog brasileiro, logo, é **OBRIGATÓRIO** que o post seja em português, caso queira postar uma versão em inglês você pode estar traduzindo e postando seu post dentro da pasta `/en/nome-do-seu-post-em-inglês`, e adicionando ele nos `relative_posts` do seu cabeçalho:

```markdown
+++
relative_posts=[
    {label="Postagem em Inglês", url="/en/nome-do-seu-post-em-inglês"}
]
+++
```

## Conclusão
Dividir conhecimento é uma das melhores maneiras de aprender e/ou aprimorar um conhecimento enquanto ajuda pessoas que ainda não portam o mesmo. Um blog, dentro de uma comunidade, é um passo importante para o fortalecimento e difusão de conhecimento dentro dela. Não esqueça de entrar no grupo de telegrama, opinar em posts abrindo ***issues*** no Github. Leia também os códigos de consulta, seguiremos ele dentro da comunidade e consecutivamente no blog: 

https://www.rust-lang.org/pt-BR/conduct.html

Abraços e até a próxima
