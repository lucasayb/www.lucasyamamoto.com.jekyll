---
layout: post
title:  "Deploy de app em Node com Docker e Elastic Beanstalk"
date:   2020-05-23 15:53:42 -0300
categories: devops
tags: node aws devops
---
![Deploy de app em Node com Docker e Elastic Beanstalk](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/46600198075_800187a13b_b.jpg)

Recentemente na [CodeBy](https://codeby.com.br) migramos nossos apps e deployments da Digital Ocean para a AWS e com isso pudemos usufruir de toda a gama de serviços que a Amazon Web Services possui. Hoje, atualmente, fazemos uso dos seguintes:

- Elastic Container Registry
- Code Pipeline
- Elastic Beanstalk
- S3

Enquanto utilizamos o S3 para servir mídias como vídeos ou imagens e hospedagem de sites estáticos, os outros são para construir nosso processo de deploy. Hoje gostaria de detalhar como montamos ele.

### Requisitos

- Conhecimento em Git e GitHub.
- Uma aplicação Node (iremos utilizar uma aplicação simples neste tutorial).
- Conhecimento básico em Docker e criação de `Dockerfile`.

## 1. Crie um repositório no GitHub

Além da migração de Digital Ocean para AWS, também fizemos a mudança de GitLab para GitHub, pelo menos para os apps que precisamos hospedar. Isso porque infelizmente a AWS não possui integração com o GitLab por padrão. Para utilizar o GitLab, foi necessário incluir neste processo o CodeCommit da AWS, mas como o objetivo era simplificar e não complicar, decidimos por utilizar o GitHub.

No GitHub, crie um repositório com as informações exigidas:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_16.24.42.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_16.24.42.png)


Com o repositório criado, vamos colocar nossa aplicação simples nele para que possamos utilizar em nosso processo. Para facilitar, deixei o repositório deste app público no GitHub. Você pode acessá-lo aqui: [https://github.com/lucasayb/aplicacao-node-simples](https://github.com/lucasayb/aplicacao-node-simples)

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_16.30.31.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_16.30.31.png)

Basicamente essa aplicação é simples. O único objetivo dela é mostrar "Hello World" na tela.

Com nossa aplicação no GitHub, podemos prosseguir para o próximo passo.

## 2. Crie uma imagem Docker de sua aplicação

Para subirmos no AWS, precisamos primeiro que nossa aplicação esteja "Dockerizada", ou seja, que ela seja transformada em uma imagem Docker para que possamos subi-la no Elastic Container Registry e buscá-la no Elastic Beanstalk.

O que precisamos é de um `Dockerfile`. Ele quem dará os passos necessários para que nossa aplicação execute corretamente. Coloque o seguinte conteúdo na raiz da sua aplicação Node e nomeie-o como `Dockerfile` (sem extensão mesmo).

```docker
FROM node:10.9.0
WORKDIR /var/www/app
COPY package.json yarn.lock ./
RUN yarn
COPY . .
EXPOSE 3000
CMD yarn start
```

### Explicando os passos do nosso `Dockerfile`

- `FROM node:10.9.0`: Aqui estamos definindo qual a imagem que será a base do nosso `Dockerfile`. Como nosso app é em Node, estamos definindo que a base da imagem que estamos criando é o `node` versão `10.9.0`, pois esta possui todas as libs necessárias para executar nossa aplicação atual.
- `WORKDIR /var/www/app`: Estamos definindo onde nossa aplicação estará dentro do nosso container Docker. Tendo em mente que o Docker cria containers com base em imagens, dentro do nosso container criado utilizando nossa imagem, nossa aplicação estará disponível em `/var/www/app`.
- `COPY package.json yarn.lock ./`: Estamos copiando os arquivos `package.json` e `yarn.lock` apenas para que possamos instalar os módulos de nossa aplicação antes de copiá-la por inteiro. O objetivo desta prática, é fazer o uso do cache do Docker, que executa o respectivo passo apenas se os arquivos utilizados neste tiverem sido alterados, e fazer uma imagem performática, com um tempo de build baixo.
- `RUN yarn`: aqui instalamos nossas libs definidas no `package.json` e `yarn.lock`.
- `COPY . .`: com nossas libs instaladas, copiamos o resto de nossa aplicação dentro da imagem.
- `EXPOSE 3000`: aqui apenas definimos que queremos expor a porta `3000`, visto que é nela que nossa aplicação estará rodando. O `EXPOSE` na realidade não serve para publicar a porta, apenas para servir como uma documentação para que o desenvolvedor possa saber qual porta deve ser de fato publicada.
- `CMD yarn start`: este é o commando que executará o container da nossa aplicação. Defini no `scripts` do `package.json` que o comando `start` é quem executará `node ./server.js`.

Feito isso, podemos testar o build de nossa aplicação localmente simplesmente rodando o seguinte commando: 

```bash
╰─$ docker build -t aplicacao-node-simples .
```

O retorno deverá ser algo parecido com isso:

```bash
Sending build context to Docker daemon  2.063MB
Step 1/7 : FROM node:10.9.0
 ---> a860762a13bc
Step 2/7 : WORKDIR /var/www/app
 ---> Using cache
 ---> ae825ecb383e
Step 3/7 : COPY package.json yarn.lock ./
 ---> e012a4029be8
Step 4/7 : RUN yarn
 ---> Running in c60f8b0ec8b6
yarn install v1.9.2
[1/4] Resolving packages...
[2/4] Fetching packages...
(node:6) [DEP0005] DeprecationWarning: Buffer() is deprecated due to security and usability issues. Please use the Buffer.alloc(), Buffer.allocUnsafe(), or Buffer.from() methods instead.
[3/4] Linking dependencies...
[4/4] Building fresh packages...
Done in 1.59s.
Removing intermediate container c60f8b0ec8b6
 ---> b6f3cae0388f
Step 5/7 : COPY . .
 ---> 316a98b3d739
Step 6/7 : EXPOSE 3000
 ---> Running in 541309c8fb87
Removing intermediate container 541309c8fb87
 ---> 567b6c4f8104
Step 7/7 : CMD yarn start
 ---> Running in 5f31defe8178
Removing intermediate container 5f31defe8178
 ---> e0eaf31d3b6e
Successfully built e0eaf31d3b6e
Successfully tagged aplicacao-node-simples:latest
```

Poderá levar um tempo, caso você não tenha a imagem do `node:10.9.0` em sua máquina.

Caso tudo tenha ocorrido conforme planejado, você poderá iniciar um container com base na imagem criada.

```bash
╰─$ docker run -e NODE_ENV=production -p 3000:3000 aplicacao-node-simples
```

Teremos o seguinte retorno e poderemos ver que nossa aplicação está rodando:

```bash
yarn run v1.9.2
$ node ./server.js
Server running at port 3000
```

Acessando [http://localhost:3000/](http://localhost:3000/), veremos nossa aplicação rodando em nossa máquina :) 

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_17.03.29.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_17.03.29.png)

Salve, e suba no GitHub.

## 3. Crie um repositório no Elastic Container Registry

Acessando o Elastic Container Registry, crie um repositório para que possamos armazenar nossas imagens. Faremos isso para que possamos buscá-la no Elastic Beanstalk:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_17.06.14.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_17.06.14.png)

Basta preencher o nome do repositório e clicar em **Create repository**.

Teremos então nossa imagem criada:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_17.09.07.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_17.09.07.png)

Guarde a informação da URI para que possamos utilizá-la mais tarde.

## 4. Crie uma aplicação e um ambiente no Elastic Beanstalk

Acesse o Elastic Beanstalk e crie um aplicativo novo.

1. Coloque o **Nome do aplicativo**:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_17.13.00.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_17.13.00.png)

2. Insira as tags do aplicativo (opcional):

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_17.14.38.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_17.14.38.png)

3. Selecione a **Plataforma** que iremos utilizar. No caso, **Docker**. E a **Ramificação da plataforma** eu costumo selecionar **Multi-container Docker**. Dessa forma, podemos adicionar novos containeres dentro do mesmo ambiente:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_17.16.39.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_17.16.39.png)

4. Em seguida, em **Código do aplicativo**, podemos deixar **Aplicativo de exemplo** selecionado, visto que este será o aplicativo padrão que iniciará com nossa aplicação. Ao invés de clicar em **Criar aplicativo** diretamente, clique em **Configurar mais opções** para personalizar como nosso ambiente será criado. Lembrando que este passo é importante pois algumas modificações não podem ser feitos quando nosso ambiente já estiver rodando:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_17.18.55.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_17.18.55.png)

5. Em **Predefinições de configuração**, selecione **Configuração personalizada**. Isso nos permitirá configurar o **Load balancer** caso queiramos e configurar o **SSL** para nosso domínio (Isso é algo que deixarei para um próximo tutorial):

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_17.22.09.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_17.22.09.png)

6. No card de **Capacidade**, clique em **Editar**. Nesta nova tela, apenas edite as instâncias de **Máx** para **1** e clique em **Salvar**:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_17.24.45.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_17.24.45.png)

7. Como tudo está configurado conforme desejamos, clique em **Criar aplicativo**. Uma nova tela se abrirá e alguns minutos levarão até que nosso aplicativo esteja rodando:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_17.27.07.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_17.27.07.png)

8. Aguarde estes minutos (algo em torno de 10 minutos) e veja a aplicação rodando na URL fornecida pelo AWS, abaixo do nome do ambiente:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_17.43.39.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_17.43.39.png)

9. Pronto! Seu ambiente foi criado! Provavelmente você terá o seguinte resultado:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_17.44.08.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_17.44.08.png)

## 5. Criando o `buildspec`

O **Buildspec** nada mais é que o arquivo responsável por saber como prosseguir neste build. Na raiz de nosso projeto, criaremos um arquivo chamado `buildspec.yml`. Nele, teremos as regras de como o CodeBuild irá prosseguir para realizar a compilação de nosso projeto:

```yaml
version: 0.2
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...          
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
artifacts:
  files:
    - Dockerrun.aws.json
```

Para facilitar, vamos "dissecar" o conteúdo da seção de `phases` deste arquivo:

```yaml
pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
```

Nesta seção, estamos nos conectando no Elastic Container Registry.

```yaml
build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...          
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
```

Aqui em `build`, estamos executando o `docker build` para que nossa imagem seja criada com base no nosso `Dockerfile` criado anteriormente e em seguida incluindo algumas tags para podermos identificar nossa imagem mais facilmente.

```yaml
post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
```

Em `post_build`, simplesmente realizamos o `docker push`, ou seja, enviamos a nossa imagem para um repositório Docker, no caso, o Elastic Container Registry.

Repare que em cada um desses passos possuímos algumas variáveis. Essas são as variáveis que serão inserida em nosso pipeline.

```yaml
artifacts:
  files:
    - Dockerrun.aws.json
```

Neste último passo, geramos um **Artefato** chamado `Dockerrun.aws.json`. Basicamente, é simplesmente o que iremos enviar para o Elastic Beanstalk. No passo seguinte, irei explicar como criar este arquivo.

Salve, e suba no GitHub.

## 6. Criando o `Dockerrun.aws.json`

O arquivo `Dockerrun.aws.json` é responsável por orientar o Elastic Beanstalk como ele deve prosseguir e o que ele irá executar. Para isso, devemos inserir as regras necessárias nele. Crie o arquivo com este exato nome na raiz de seu projeto, com o seguinte conteúdo:

```json
{
    "AWSEBDockerrunVersion": 2,
    "volumes": [],
    "containerDefinitions": [
      {
        "name": "app",
        "image": "616465875746.dkr.ecr.us-east-1.amazonaws.com/aplicacao-node-simples:latest",
        "environment": [],
        "essential": true,
        "memory": 256,
        "links": [],
        "mountPoints": [
          {
            "sourceVolume": "awseb-logs-app",
            "containerPath": "/app/log"
          }
        ],
        "portMappings": [
          {
            "containerPort": 3000,
            "hostPort": 80
          }
        ]
      }
    ]
  }
```

Em `image`, colocaremos a URI gerada pelo Elastic Container Registry com a tag `latest`, definida no passo do CodePipeline. Em `portMappings`, simplesmente iremos inserir em `containerPort`, a porta na qual nossa aplicação irá rodar, no caso, `3000`.

Salve, e suba no GitHub.

## 7. Crie um pipeline no CodePipeline

Agora, para colocarmos nossa aplicação no ar, iremos criar um novo pipeline. Um pipeline nada mais é que uma série de passos que automatizam o processo de build e deploy do nosso app, conhecido como CD, ou, Continuous Deployment.

1. Insira o nome de seu pipeline. Para que nosso pipeline tenha acesso aos serviços necessários para o deploy, ele criará automaticamente uma função de serviço. Iremos precisar alterar esta função de serviço mais tarde. Em seguida, clique em **Próximo**:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_17.47.08.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_17.47.08.png)

1. Aqui teremos as opções de provedor de origem, ou seja, de onde nosso código irá ser extraído para realizar esse processo. Como criamos um repositório no GitHub mais cedo, selecione **GitHub** em **Provedor de origem**. Clique em **Conectar ao GitHub** , e assim que você se logar no GitHub, selecione o repositório desejado e branch que irá disparar esse pipeline. Basicamente, toda vez que um push for disparado em nossa branch, no meu caso `master`, o processo do pipeline irá se iniciar. Em seguida, clique em **Próximo**:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_17.53.18.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_17.53.18.png)

1. Na etapa de compilação, selecione o **AWS CodeBuild** como provedor de compilação. Em seguida, clique em **Criar projeto**. Uma nova aba irá se abrir. Nessa aba, na seção de **Configuração do projeto**, insira o **Nome do projeto**:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_17.55.15.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_17.55.15.png)

4. Na seção de **Ambiente**, mantenha selecionado **Imagem gerenciada** para a **Imagem de ambiente:**

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_18.02.19.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_18.02.19.png)

5. Coloque o **Sistema operacional** como **Ubuntu. Tempo(s) de execução** selecionaremos **Standard**. **Imagem** selecionaremos **aws/codebuild/standard:1.0**. **Versão da imagem**, selecionaremos **Usar sempre a imagem mais recente para esta versão do tempo de execução** e em **Tipo de ambiente** manteremos selecionado **Linux**. Habilite a opção de **Privilegiado** pois nosso pipeline irá gerar uma imagem. Perceba também que esse build irá criar uma nova função de serviço. Guarde-a também pois iremos precisar modificá-la mais tarde:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_18.03.16.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_18.03.16.png)

6. Na seção de **Buildspec**, manteremos selecionada a opção de **Usar um arquivo buildspec**. Este é o arquivo que criamos anteriormente.

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_18.04.48.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_18.04.48.png)

7. Na seção de **Logs**, manteremos as configurações selecionadas e para finalizar, clicaremos em **Continuar para CodePipeline**.

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_18.26.45.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_18.26.45.png)

8. De volta no passo de build do CodePipeline, na seção **Variáveis de ambiente**, insira as seguintes variáveis:

- `AWS_DEFAULT_REGION`: a sua região do AWS. No meu caso, `us-east-1`
- `AWS_ACCOUNT_ID`: seu id de conta. Ele está presente na sua URL gerada pelo Elastic Container Registry
- `IMAGE_REPO_NAME`: nome do seu repositório criado no Elastic Container Registry. No meu caso, **aplicacao-node-simples**.
- `IMAGE_TAG`: a tag de sua imagem. Podemos deixar `latest` como padrão.

Você deve ter algo conforme o seguinte:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_18.33.35.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_18.33.35.png)

Clique em **Próximo** e prossiga.

9. Este é o passo de **Implantação**. É nele que enviaremos nossas informações da nossa aplicação para o Elastic Beanstalk. Em **Provedor de implantação**, selecione **AWS Elastic Beanstalk**, **Nome do aplicativo** selecione o aplicativo criado no EBS e em **Nome do ambiente**, selecione o ambiente criado no EBS, juntamente com nosso aplicativo:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_18.36.35.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_18.36.35.png)

10. Revise suas informações, e caso tudo esteja correto, clique em **Criar pipeline**.

11. Logo em seguida, seu pipeline já será executado. Provavelmente o passo de **Build** irá falhar.

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_18.40.30.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_18.40.30.png)

Visualizando os detalhes, poderemos ver o seguinte erro:

```bash
An error occurred (AccessDeniedException) when calling the GetAuthorizationToken operation: User: arn:aws:sts::616465875746:assumed-role/codebuild-aplicacao-node-simples-service-role/AWSCodeBuild-cec2e87f-49bc-41cb-b95f-29930a6b74fc is not authorized to perform: ecr:GetAuthorizationToken on resource: *
```

Significa que o o CodeBuild não possui autorização para subir nossa imagem ao Elastic Container Registry. Para corrigir isso, vamos alterar as funções de serviço, criadas anteriormente pelo CodePipeline e pelo CodeBuild, para que eles possam ter as permissões necessárias no processo. Acesse o AWS IAM. Na barra lateral, clique em **Funções** e busque pela primeira função criada no processo do CodeBuild, no meu caso, `codebuild-aplicacao-node-simples-service-role`: 

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_18.44.58.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_18.44.58.png)

Clique na função, e em seguida, clique em **Anexar políticas**:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_18.45.35.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_18.45.35.png)

Na tela seguinte, busque por Elastic Container Registry e selecione a política **AmazonEC2ContainerRegistryPowerUser**. Basicamente, essa política é responsável por escrever e ler imagens no Elastic Container Registry, porém ela não poderá deletar nenhuma imagem. Clique em **Anexar política**: Faça o mesmo para a função criada pelo Code Pipeline, no meu caso, `AWSCodePipelineServiceRole-us-east-1-aplicacao-node-simples`:

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_18.46.19.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_18.46.19.png)

Altere também a função `aws-elasticbeanstalk-ec2-role` para que o Elastic Beanstalk possa ter acesso ao Elastic Container Registry. Isso deverá ser feito apenas uma vez.

Feito isso, retorne ao Code Pipeline, selecione seu pipeline criado e clique em **Lançar alteração**. Basicamente, ele irá executar o pipeline baseado no último commit do seu repositório.

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_18.50.21.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_18.50.21.png)

Caso tudo tenha sido feito corretamente, seu pipeline deve ser executado agora sem problemas!

Acesse sua aplicação e *voilà*! 

![Deploy de app em Node com Docker e Elastic Beanstalk/Screen_Shot_2020-05-23_at_19.02.37.png](/assets/images/2020-05-23-deploy-de-app-em-node-com-docker-e-elastic-beanstalk/Screen_Shot_2020-05-23_at_19.02.37.png)

> Importante ressaltar que todo este processo está utilizando recursos do AWS. Isso gera custos. Caso sua intenção seja apenas para aprendizado, não esqueça de deletar os recursos utilizados, se não cobranças indesejadas serão emitidas!