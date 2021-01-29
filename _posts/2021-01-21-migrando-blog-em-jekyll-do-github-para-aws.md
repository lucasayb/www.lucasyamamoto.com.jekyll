---
layout: post
title:  "Migrando blog em Jekyll do GitHub para AWS"
date:   2021-01-21 15:53:42 -0300
categories: devops
tags: devops jekyll github
redirect_from: 
  - /devops/2020/05/23/migrando-blog-em-jekyll-do-github-para-aws.html
---
![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/jekyll-og.jpg](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/jekyll-og.jpg)


Estou a um tempo sem postar no meu blog, mas como estou em busca de tirar a certificação de AWS, e decidi por usar AWS em todos os lugares que eu puder, incluindo meu blog!

Hoje quero migrar meu blog em Jekyll para o AWS, mais especificamente, o S3.

Como vocês já devem ter percebido, até o momento em que escrevo esse post meu blog está hospedado no GitHub Pages, algo que tem seus benefícios, como hospedagem grátis, mas eventualmente traz alguns contras como o fato de meu repositório estar público para qualquer pessoa copiar o código. Claro que não tenho nenhum dado sensível neste repositório, visto que todos os posts são públicos, mas futuramente pretendo transformar este blog em algo maior, com uma arquitetura mais planejada e contendo mais serviços e componentes.

Chega de enrolação e vamos nessa!

## Criando o bucket no S3

Para que possamos hospedar o blog, precisamos utilizar o S3, e com isso, necessitamos de um bucket onde armazenaremos os dados estáticos gerados pelo Jekyll.

No console da Amazon, acesse S3 e clique em "**Criar bucket**"

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_20.53.34.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_20.53.34.png)

Nesta seção, preencha as informações do seu novo bucket

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_20.53.59.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_20.53.59.png)

No nome do bucket, vou colocar "lucasyamamoto.com", visto que este nome deve ser único no S3 inteiro. E vou mantê-lo em us-east-1, visto que é a região mais barata.

Durante a criação do bucket, devemos desativar a opção "Bloquear acesso público", visto que queremos que todos sejam capazes de acessar nosso blog. E também, por questões de segurança, a AWS exige que você marque a opção "Desativar o bloqueio de todo o acesso público pode fazer com que este bucket e os objetos dentro dele se tornem públicos"

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_20.54.34.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_20.54.34.png)

Para fazer o controle dos recursos, vou adicionar uma tag também

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_20.55.45.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_20.55.45.png)

Depois é só clicar em "Criar bucket".

Em seguida, com o bucket criado, acesse o bucket e acesse a aba de "Propriedades", na qual você deverá buscar o painel "Hospedagem de site estático" e clicar em "Editar" nele.

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_20.57.31.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_20.57.31.png)

Nesta nova página aberta, ative a hospedagem de sites estáticos e preencha a opção "Documento de índice" com "index.html" e "Documento de erro" com "404.html", sendo que essas são páginas que constarão na versão final do build gerado pelo Jekyll

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.13.03.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.13.03.png)

Em seguida clique em "Salvar alterações".

E para finalizar a configuração do nosso bucket, vá na aba "Permissões" do bucket e clique em "Editar" na política do bucket.

Você deve colocar o seguinte conteúdo nesta página:

```json
{
    "Version": "2012-10-17",
    "Id": "Policy1611273646861",
    "Statement": [
        {
            "Sid": "Stmt1611273643203",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion"
            ],
            "Resource": "arn:aws:s3:::lucasyamamoto.com/*"
        }
    ]
}
```

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.01.18.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.01.18.png)

Basicamente estamos abrindo nosso bucket para leitura pública, ou seja, todos poderão ver os dados estáticos desse bucket, visto que é onde estarão os dados estáticos do nosso site Jekyll.

Bom, agora com nosso bucket configurado, podemos seguir para o próximo passo!

## Deploy contínuo

Uma das vantagens que não pretendo perder nessa migração é o fato de um push resultar em um deploy. Com isso, sempre que lanço alguma alteração no blog como um post ou uma modificação bem objetiva, faço um commit, um push e o deploy é feito automaticamente pelo GitHub Pages. Podemos fazer isso também com o CodePipeline.

Acessando o CodePipeline, podemos começar a criação de um novo Pipeline.

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.24.44.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.24.44.png)

Ao clicar em "Criar pipeline", devemos dar um nome a ele. Podemos manter as opções selecionadas por padrão e clicar em "Próximo".

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.25.04.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.25.04.png)

Na primeira seção do nosso pipeline, que é a de origem, devemos colocar de onde sairá o código que será implantado por este fluxo, sendo neste caso do meu GitHub. Basta apenas se conectar com o GitHub, selecionar o repositório desejado e a branch, e por final clicar em "Próximo".

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.25.59.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.25.59.png)

Na etapa de compilação, devemos selecionar o "Provedor de compilação", sendo neste caso o CodeBuild, visto que é através dele que daremos as instruções de como ele deverá realizar o build de nosso código Jekyll.

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.48.08.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.48.08.png)

Clique em "Criar projeto", e na página que abrir, preencha as informações.

A primeira informação é o "Nome do projeto".

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.27.21.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.27.21.png)

Em seguida, siga as seguintes instruções para configurar o ambiente de build:

- Sistema operacional: Ubuntu
- Tempo(s) de execução: Standard
- Imagem: aws/codebuild/standard:3.0
- Versão da imagem: Usar sempre a imagem mais recente para esta versão do tempo de execução
- Tipo de ambiente: Linux
- Ative a opção de acesso privilegiado, para que possamos rodar comandos como root.

E para finalizar, clique em "Criar projeto"

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.27.39.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.27.39.png)

Com a etapa de compilação (build) configurada, podemos seguir para a etapa de implantação, ou seja, a etapa de que fará o deploy propriamente dito.

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.49.30.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.49.30.png)

Nesta etapa, devemos selecionar o Amazon S3 como provedor de implantação e selecionar o bucket que nós criamos anteriormente. Não esqueça de marcar a opção "Extraia o arquivo antes de implantar", visto que nosso build irá gerar um zip.

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.49.54.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.49.54.png)

Revise as informações e crie o pipeline.

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.50.01.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_21.50.01.png)

## Configurando nosso repositório

Para finalizar nosso processo, devemos adicionar um arquivo `buildspec.yml`, contendo o seguinte conteúdo:

```yaml
version: 0.2
env:
  variables:
    JEKYLL_ENV: production
phases:
  install:
    runtime-versions:
      ruby: 2.6
    commands:
      - gem install bundler
      - bundle install
  build:
    commands:
      - bundle exec jekyll build
artifacts:
  type: zip
  base-directory: _site
  files:
    - '**/*'
```

Esse arquivo é responsável por ditar as instruções ao nosso pipeline para que ele possa saber como deverá gerar nossos arquivo.

```yaml
version: 0.2
```

Esta é a versão do nosso arquivo. Cada arquivo tem uma versão no AWS para que com o lançamento de novas features não impacte os fluxos já existentes em uma versão x.

```yaml
env:
  variables:
    JEKYLL_ENV: production
```

Aqui definimos as variáveis do ambiente de build. No caso, temos apenas a `JEKYLL_ENV` na qual definiremos o ambiente como `production`.

```yaml
phases:
  install:
    runtime-versions:
      ruby: 2.6
    commands:
      - gem install bundler
      - bundle install
  build:
    commands:
      - bundle exec jekyll build
```

Aqui em `phases`, temos duas partes.

- `install` é responsável por fazer o processo de configuração do ambiente de build, instalando o `bundler` , visto que é o gerenciador de pacotes para Ruby, linguagem na qual o Jekyll é escrito, e rodamos o `bundle install` para que possamos instalar os outros pacotes (gems) que constam no nosso `Gemfile`.  Dessa forma, teremos o Jekyll instalado e configurado corretamente.
- `build` é o processo que gera os arquivos estáticos. Esses arquivos são os que serão acessados pelos nossos usuários e os que estarão armazenados no S3.

```yaml
artifacts:
  type: zip
  base-directory: _site
  files:
    - '**/*'
```

Nesta parte final, estamos apenas retornando nossos artefatos, que para o CodePipeline são os arquivos resultados de qualquer etapa do fluxo. No caso, queremos apenas a pasta `_site`, visto que é ela quem irá conter nossos arquivos estáticos que serão enviados para o S3.

Para finalizar, podemos apenas realizar o commit desse código com o nome de `buildspec.yml` na raiz de nosso repositório e dar um push!

Observando nosso pipeline, podemos ver que ele foi executado com êxito!

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_22.17.57.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_22.17.57.png)

Para que possamos acessar o blog, podemos ir no nosso bucket e selecionarmos a aba "Propriedades", e em seguida buscarmos a URL do nosso bucket que consta em "Endpoint de site de bucket".

![/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_22.19.22.png](/assets/images/2021-01-21-migrando-blog-em-jekyll-do-github-para-aws/Screen_Shot_2021-01-21_at_22.19.22.png)

Acessando o blog, podemos ver que nosso fluxo funcionou!

[]()