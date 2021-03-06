---
layout: post
title: Commits automáticos no GitHub com Travis CI!
tags: [git, github, gh-pages, travis-ci]
---

O [GitHub] é um serviço web para hospedagem, gestão e compartilhamento
de repositórios [Git] que oferece diversos recursos, entre eles as
[GitHub Pages] e intregração contínua, com [Travis CI], por exemplo. As
GitHub Pages são páginas web gerenciadas e renderizadas pelo próprio
GitHub para que seus usuários possam exibir seus projetos de uma forma
simples. Já a integração contínua com Travis realiza, essencialmente,
teste de código. Em pacotes R, por exemplo, podemos ter o status do
pacote que mostra se um *push* para o repositório ainda o deixa sem
erros, mas também podemos fazer mais com esse serviço, como veremos a
seguir.

Para criar uma GitHub Page é só criar um ramo com nome `gh-pages` em seu
projeto e adicionar a este arquivos com extensão `.html`. Além dessa
forma simples de criação, pode-se escolher diversos templates [Jekyll]
(veja [link]), pois as páginas são geradas pelo Jekyll. Neste post será
abordada apenas a primeira opção. Já para utilizar o Travis CI, basta
conectar-se ao serviço, pode ser através de seu conta no GitHub,
habilitar a integração contínua ao seu repositório e incluir um arquivo
`.travis.yml` com as instruções que o servidor Travis deve seguir.

Comumente nós, usuários de R, utilizamos o pacote [`rmarkdown`] para
escrevermos nossos arquivos em MarkDown mesclando com códigos R e
compilando para o formato `.hmtl`. Pensando em uma rotina para criação
das GitHub Pages, teríamos nosso ramo `master` com o projeto e um ramo
`gh-pages` onde deixamos todos os `.html` e demais arquivos gerados da
compilação. Todavia esse trâmite de migrar entre ramos, compilar os
arquivos, commitar os arquivos gerados pela compilação, etc. é
demasiadamente entediante e cansativo. Então porque não automatizar?

Para automatizar esse processo, de inclusão de arquivos gerados da
compilação no ramo `gh-pages`, vamos utilizar o serviço Travis CI
conforme os passos a seguir:

**Habilite a integração contínua no seu repositório**

No site do [Travis CI], serão listados todos os repositórios do GitHub
dos quais se tem acesso. Habilite a integração contínua, conforme figura
abaixo.

![](https://raw.githubusercontent.com/JrEduardo/jreduardo.github.io/master/_posts/images/travis1.png)
<!-- ![](images/travis1.png) -->

**Gere uma chave de acesso ao seu repositório**

No GitHub há as permissões de acesso, isso faz com que pessoas não
autorizadas não possam modificar um repositório sem permissão. Como o
servidor Travis é análogo à um usuário sem acesso, devemos dar-lhe
permissão para escrita no repositório. Isso é feito na página de
*Configurações -> Personal access tokens*.

![](https://raw.githubusercontent.com/JrEduardo/jreduardo.github.io/master/_posts/images/github1.png)
<!-- ![](images/github1.png) -->

Ainda nessa página, ao criar uma nova chave de acesso, você escolherá o
escopo que essa chave compreende. Para incluir os arquivos em um ramo
apenas a primeira opção é necessária (na verdade apenas o terceiro item
da primeira opção já será suficiente).

![](https://raw.githubusercontent.com/JrEduardo/jreduardo.github.io/master/_posts/images/github2.png)
<!-- ![](images/github2.png) -->

Concedida as permissões à chave, uma página que contém o código
identificador MD5 dessa chave é gerado. Copie esse código, note o aviso,
pois ele é verdadeiro, você não conseguirá visualizar novamente o código
pelo GitHub.

![](https://raw.githubusercontent.com/JrEduardo/jreduardo.github.io/master/_posts/images/github3.png)
<!-- ![](images/github3.png) -->

**Crie uma variável de ambiente no Travis**

O servidor Travis, ao executar a verificação do seu repositório, têm
várias variáveis que são acessadas durante a verificação. Nesta etapa
fazemos com que o Travis conheça a chave de acesso que geramos na etapa
anterior.

![](https://raw.githubusercontent.com/JrEduardo/jreduardo.github.io/master/_posts/images/travis2.png)
<!-- ![](images/travis2.png) -->

**Adicione os arquivos para compilação dos documentos**

Agora já foram realizadas todas as etapas para que o Travis tenha acesso
ao nosso repositório. Todavia ainda lhe falta a instrução sobre o que
fazer neste repositório. Tome a estrutura de um repositório que já tenha
um ramo `gh-pages` e um ramo `master` conforme abaixo:

```bash
.
├── page-files
│   ├── cap01.Rmd
│   ├── cap02.Rmd
│   ├── cap03.Rmd
│   └── index.Rmd
└── README.md
```

Pois bem, listando os afazeres do Travis temos:

1. Compilar os arquivos `.Rmd`.\
   A sugestão aqui é deixar os arquivos que fazem parte da sua página
   web em um diretório específico. Para facilitar os procedimentos
   posteriores. Para compilar esses arquivos sugere-se a criação de um
   script `.R`, por exemplo `_render.R` contendo o código:

```r
## Compila os Rmd

dirname <- "./page-files"
files <- grep(".Rmd$", dir(dirname), value = TRUE)
sapply(paste0(dirname, "/", files), rmarkdown::render)
```

2. Commitar as mudanças no ramo `gh-pages`.\
   Como o serviço Travis CI faz a verificação com servidores Linux, um
   script `bash` é ideal para automatizar a rotina de commits. Em um
   arquivo `_deploy.sh` faça:

```bash
#!/bin/sh

git config --global user.email "edujrrib@gmail.com"
git config --global user.name "Travis boot"

git clone -b gh-pages https://${GIT_KEY}@github.com/${TRAVIS_REPO_SLUG}.git dir-tmp
cd dir-tmp
cp -r ../page-files/* ./
git add --all *
git commit -m "Atualização automática (travis build ${TRAVIS_BUILD_NUMBER})" || true
git push origin gh-pages
```

Note que as porções do código com a sintaxe `${variavel}` são as
variáveis de ambiente do Travis CI. A `GIT_KEY` refere-se ao código de
acesso criado nas etapas anteriores, as demais variáveis são as criadas
automaticamente pelo Travis.

**Adicionar as instruções para o Travis**

A verificação do Travis é realizada a partir das instruções colocadas em
um arquivo `.travis.yml`. Esse arquivo permite várias linguagens e
várias definições, mais sobre como instruir o Travis a partir deste
arquivo pode ser visto [aqui]. No nosso caso o seguindo conteúdo será
colocado nesse arquivo.

```yaml
language: r

before_script:
  - chmod +x ./_deploy.sh

script:
  - Rscript -e "source('_render.R')"
  - ./_deploy.sh
```

Em ordem esse arquivo diz:

1. A linguagem utilizada é R
2. Antes do início da verificação é permita a execução do script bash
3. O script de verificação:
     - Execute o `_render.R` (gerando todos os arquivos para a página
       web).
     - Execute o `_deploy.sh` (clonando o repositório, adicionando os
       arquivos gerados e commitando-os).

Outro detalhe é que definindo a linguagem R, o Travis procurará um
arquivo `DESCRIPTION` para que se baixe as dependências necessárias para
verificação, isso é feito pois a linguagem R no Travis é usada
majoritariamente para verificação de pacotes R. Portanto no DESCRIPTION
fazemos:

```
Package: qualquer_nome
Title: Qualquer Titulo
Version: 0.0.1
Imports: rmarkdown
```

Agora já podemos ficar despreocupados com a geração dos arquivos que
farão parte da nossa página web, pois o Travis fará todo o trabalho
sujo. Abaixo exibi-se a página do Travis após o build do repositório
[book-test], que seguiu as especificações deste post.

<!-- ![](https://raw.githubusercontent.com/JrEduardo/jreduardo.github.io/master/_posts/images/travis3.png) -->
![](images/travis3.png)

E abaixo temos a página inicial do ramo `gh-pages` desse
repositório. Note que a mensagem de commit é a qual definimos no arquivo
`_deploy.R`.

![](https://raw.githubusercontent.com/JrEduardo/jreduardo.github.io/master/_posts/images/github4.png)
<!-- ![](images/github4.png) -->

A página web gerada pelo GitHub fica hospedada no endereço
`https://usuario.github.io/repositorio`. A página criada para esse
tutorial pode ser acessada em https://jreduardo.github.io/book-test/
(esse repositório sofrerá alterações, pois o usarei para outros testes).

Ressalto que o material deste post foi motivado a partir da minha
curiosidade sobre o funcionamento do pacote [`bookdown`] e muito do que
foi descrito aqui pode ser explorado também nos repositório Git do
pacote ([rstudio/bookdown]) e da demostração do pacote
([rstudio/bookdown-demo]).

<!--------------------------------------------- -->
[GitHub]: https://github.com/
[Git]: https://git-scm.com/
[GitHub Pages]: https://pages.github.com/
[Travis CI]: https://travis-ci.org/
[Jekyll]: https://jekyllrb.com/
[link]: http://jekyllthemes.org/
[`Rmarkdown`]: http://rmarkdown.rstudio.com/
[aqui]: https://docs.travis-ci.com/user/getting-started/
[book-test]: https://github.com/JrEduardo/book-test
[`bookdown`]: https://bookdown.org/
[rstudio/bookdown]: http://github.com/rstudio/bookdown/
[rstudio/bookdown-demo]: http://github.com/rstudio/bookdown-demo/
