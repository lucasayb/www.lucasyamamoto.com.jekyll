---
layout: post
title:  "Migrando blog em Jekyll do GitHub para AWS"
date:   2021-01-28 17:38:02 -0300
categories: desenvolvimento
tags: devops dicas docker
---
![/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/featured.jpg](/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/featured.jpg)

Quando se trabalha com diversas tecnologias e se possui pouco espaço em seu computador, precisamos recorrer a algumas alternativas mais práticas para que possamos ter algo minimamente viável, se tratando de entregas ágeis, e econômico.

Atualmente utilizo um MacBook i5 2017 para programar e um dos principais problemas dele hoje é o fato de ele possuir 128gb. Hoje meus dados estão na nuvem e todos os meus projetos pessoais e da empresa são versionados e salvos no GitHub ou no GitLab, o que seria ótimo se eu trabalhasse apenas com uma tecnologia. Porém este não é o caso. Já fiz projetos em Ruby On Rails, NodeJS, Python com Django ou Flask, utilizando React, e várias outras tecnologias diferentes, e todas elas possuem algo em comum: o armazenamento das bibliotecas de cada gerenciador de pacotes.

Supondo que em um dia eu precise fazer um projeto que utilize React, vou precisar salvar gerenciar meus pacotes com o `npm`. A versão do meu JavaScript deve ser gerenciada com o `nvm`. E isso se tratando apenas de front end. Indo para o back end, gostaria de utilizar Ruby On Rails. Meu gerenciador de pacotes seria o `bundler`, e o gerenciador das versões do Ruby seria o `rvm`. Em pouco tempo, podemos atingir esse "teto" de 128gb.

Diante de tantas soluções existentes, como utilizar uma instância EC2 no AWS para cuidar desses projetos, ou até mesmo usar algo como o Cloud 9, decidi seguir pelo caminho mais barato e prático (para mim, é claro).

## Instalando o Docker

Já trabalho há um bom tempo com Docker, visto que atualmente é como subo os meus sistemas e os da [CodeBy](https://codeby.com.br/), portanto vou mostrar como realizar a instalação para deixar esse post mais completo.

### Instalando o Docker no Mac

Instalar o Docker no Mac é algo bem direto para falar a verdade. Basta fazer o download do mesmo no Docker Hub.

[https://hub.docker.com/editions/community/docker-ce-desktop-mac/](https://hub.docker.com/editions/community/docker-ce-desktop-mac/)

E fazer o famoso processo de drag and drop do ícone do app para a pasta `applications` do seu Mac OS.

![/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/docker-app-drag.png](/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/docker-app-drag.png)

E abrindo a aplicação, o ícone do Docker aparecerá na barra de tarefas.

![/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/Screen_Shot_2021-01-22_at_16.25.36.png](/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/Screen_Shot_2021-01-22_at_16.25.36.png)

Bem fácil, não?

### Instalando o Docker no Windows

Não sou um cara muito fã de Windows, mas tenho que admitir que o processo de instalação nele é tão fácil quando no Mac. Basta baixar o Docker Desktop no Docker Hub [https://hub.docker.com/editions/community/docker-ce-desktop-windows/](https://hub.docker.com/editions/community/docker-ce-desktop-windows/) e rodar o executável. Siga as instruções do instalador e, de acordo com a documentação do Docker, certifique-se de que a opção "Enable Hyper-V Windows Features" está marcada (sinceramente não sei o que faz essa opção). 

> Se sua conta de administrador for diferente de sua conta de usuário, você deve adicionar o usuário ao grupo **docker-users**. Execute o **Gerenciamento do computador** como administrador e navegue até **Usuários e grupos locais** > **Grupos** > **docker-users**. Clique com o botão direito para adicionar o usuário ao grupo. Faça logout e login novamente para que as alterações tenham efeito.

### Instalando o Docker no Linux (Ubuntu)

Vou mostrar aqui a instalação no Ubuntu visto que sempre utilizei distros baseadas em Debian no meu tempo de Linux, mas no site do [Docker](https://docs.docker.com/engine/install) você encontra o processo de instalação para outras distros, sendo todos bem parecidos.

Diferentemente do Mac e Windows, aqui você não instala o Docker Desktop. O que você instala é apenas a engine do Docker. No final, o resultado acaba sendo o mesmo, o que muda mais é que o Docker Desktop possui uma interface para gerenciamento dos recursos.

De acordo com o site do Docker, você deve desinstalar todas as versões antigas do Docker antes de instalar a versão mais recente através do terminal. Apesar de presumir que você ainda não tenha o Docker instalado no seu sistema, basta apenas executar o seguinte comando para realizar este processo.

```bash
$ sudo apt-get remove docker docker-engine docker.io containerd runc
```

A forma mais fácil de se instalar o Docker no Linux é utilizando o script que automatiza todo o processo de download, adição de repos no seu Linux para atualizações automáticas e instalação propriamente dita. 

Para baixar e executar o script, utilize o seguinte comando.

```bash
$ curl -fsSL https://get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh
```

E para evitar precisar utilizar o `sudo` a cada comando rodado no Docker, adicione seu usuário ao grupo de usuários `docker`

```bash
$ sudo usermod -aG docker <seu-usuario>
```

Existem outros processos para instalação do Docker no Linux, mas esse é o mais fácil atualmente.

## Configurando nosso ambiente de desenvolvimento

Uma das ferramentas que realmente revolucionou o mundo dos devs é o VSCode. Como tudo que utilizamos para desenvolver em diversas tecnologias diferentes, existe um plugin para o que queremos fazer.

A ideia principal aqui é fazer um container para cada projeto. Ou seja, se vamos fazer um projeto React, devemos criar um container com uma imagem baseada em Node. E se vamos fazer um projeto em Ruby on Rails, nosso container será baseado em Ruby.

Daqui a pouco darei mais detalhes de como isso funciona na prática, vamos antes apenas instalar a extensão que orquestra essa mágica.

### [Remote - Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)

"Remote - Containers" é uma extensão desenvolvida pela própria Microsoft, o que faz com que ela seja ainda mais incrível.

O fluxo para utilizá-la é simples: após realizar a instalação dela, digite cmd + shift + P (ou ctrl + shift + p), e busque por `Remote-Containers`.

Nessa busca, você terá muitas opções interessantes. A primeira que você deverá utilizar é "Add Development Container Configuration Files..."

![/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/Screen_Shot_2021-01-22_at_17.02.22.png](/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/Screen_Shot_2021-01-22_at_17.02.22.png)

Assim que selecioná-la, uma lista de opções irá se abrir para selecionar a tecnologia na qual seu container será baseado. Normalmente, já deverá aparecer em primeiro a tecnologia principal encontrada no seu projeto com base no seu código, mas caso não seja a correta, clique em "Show all definitions" no final da lista e busque a tecnologia desejada.

![/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/Screen_Shot_2021-01-22_at_17.00.57.png](/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/Screen_Shot_2021-01-22_at_17.00.57.png)

Assim que selecionar a tecnologia, três arquivos serão criados dentro de uma pasta chamada `.devcontainer`.  Esses arquivos são instruções indicando como o Docker deve lidar com a instalação e gerenciamento das tecnologias e como ele deverá executar sua aplicação. No geral, são instruções básicas de instalação da tecnologia em questão.

![/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/Screen_Shot_2021-01-22_at_17.16.30.png](/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/Screen_Shot_2021-01-22_at_17.16.30.png)

Em seguida, o próximo comando executado deverá ser o "Open folder in container".

![/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/Screen_Shot_2021-01-22_at_17.15.29.png](/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/Screen_Shot_2021-01-22_at_17.15.29.png)

Basicamente este comando irá executar o seu projeto em um container Docker, o que nos permite instalar as bibliotecas desejadas, configurar nosso ambiente totalmente isolado e por fim, limpá-lo para não ocupar muita memória em nosso sistema.

Ao selecionar a opção acima, o Docker irá baixar a imagem base do Docker Hub, realizar o build do container com base no `Dockerfile` que consta na pasta `.devcontainer` e por fim executá-lo. Isso leva alguns minutos na primeira execução. Mas o processo é esse!

![/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/Screen_Shot_2021-01-22_at_17.21.31.png](/assets/images/2021-01-28-montando-um-ambiente-de-desenvolvimento-com-docker/Screen_Shot_2021-01-22_at_17.21.31.png)

Com o ambiente configurado, você deverá ter acesso ao terminal do container e tudo irá funcionar da forma que você está acostumado!

E no final, caso queira limpar seus containers antigos no Docker, basta executar no terminal fora de qualquer container o seguinte comando:

```bash
$ docker system prune
```

Dessa forma, você garante que os containers que não estão sendo executados atualmente serão limpos!

Embora o próprio Docker já ocupe bastante espaço no meu Mac, isso me ajuda a ter um controle maior e saber quanto de fato estou usando. E diga-se de passagem, torna o processo bem prático para cada projeto!