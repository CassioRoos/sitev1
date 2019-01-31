---
layout: post
title: "Dockerizando sua aplicação"
categories:
  - pt-br
tags:
  - pt-br
  - python
  - docker
featured-img: docker
last_modified_at: 2019-01-31T21:33:32-05:00
---


## [Docker](https://www.docker.com/get-started)

Docker é uma ferramente designada a tornar mais fácil criar, fazer deploy e rodar aplicações usando container. Containers permitem ao desenvolvedor empacotar toda uma aplicação com as suas dependências e enviar como um pacote único. Fazendo isso, o desenvolvedor pode ficar tranquilo que a aplicação irá rodar independentemente da outra máquina, desconsiderando qualquer configuração que ela possa ter diferente da máquina de origem onde foi escrito e testado.

## [Containers](https://opensource.com/resources/what-docker)

De uma certa maneira, Docker é parecido com uma VM. Mas diferente dessa, no lugar de criar um ambiente virtual completo, o Docker pertime usar o mesmo Kernel linux que o sistema está rodando e precisa somente que a aplicação seja enviada com o que ela precisa para rodar. Isto gera um grande aumento de performace e reduz o tamanho da aplicação.

## Mas por que utilizar docker na apliacação

No ano passado (2018) fui apresentado ao Docker e suas facilidades. Antes, quando precisava enviar qualquer aplicação a alguém era necessário enviar muita coisa. Containerização trouxe uma grande facilidade, tudo que preciso fazer é criar uma imagem da aplicação, subir para o dockerhub e **voilà**. Agora funciona em outras máquinas além da minha. Vou escrever ainda mais sobre docker e suas funcionalidades, mas acredito que esse post já é de grande utilidade.

## Ambiente
 
Eu utilizo o [docker no windows](https://docs.docker.com/docker-for-windows/) e ele funciona tão bem quando no linux. A instalação é bem tranquila, aquela bem padrão, NEXT³ e finish.

Será necessário habilitar o Hyper-V, que é a plataforma de processamento de virtualização.

<img src="https://i.imgur.com/ute4eU2.png" style="height:300px;"/>

Caso a opção esteja indiponível, será necessário habilitar a virtualização na BIOS. [Esse passo a passo](https://www.qnap.com/pt-pt/how-to/faq/article/como-ativar-o-intel-vt-x-e-amd-svm/) da uma idéia de como executar esse procedimento.

Depois de tudo instalado, o ícone do Docker vai aparecer na sua barra de ferramentas, aí é fazer login e ser feliz.

<img src="https://i.imgur.com/dJ6lzEF.png" style="height:300px;"/>

Para ter certeza que tudo deu certo, eu sempre executo:

```
docker run hello-world
```

Se tudo correu bem, teremos a resposta:

```
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

# Bora subir nosso container

Fiz um novo [branch do projeto](https://github.com/CassioRoos/python-base-project/tree/docker-app) para fazermos essa atividade.

Mãos a obra. Para isso vamos criar um **DockerFile**, contendo tudo que precisamos para a nossa aplicação rodar. Tentando simplificar, **DockerFile** é uma receita de bolo que contém tudo que nossa aplicação precisa para rodar. O docker vai usar essa receita para criar o bolo, que é o nosso container e depois disso ele já está pronto para uso. [**Aqui**](https://docs.docker.com/engine/reference/builder/) tem a documentação completa.

Abaixo algumas das possibilidades do DockerFile:

 - FROM: Informa a partir de qual imagem será gerada a nova imagem, lembrando que em poucos casos (Veremos em posts futuros), uma imagem será gerada se um imagem base;
MAINTAINER: Campo opcional, que informa o nome do mantenedor da nova imagem;
 - RUN: Especifica que o argumento seguinte será executado, ou seja, realiza a execução de um comando;
 - CMD: Define um comando a ser executado quando um container baseado nessa imagem for iniciado, esse parâmetro pode ser sobrescrito caso o container seja iniciado utilizando alguma informação de comando, como: docker run -d imagem comando, neste caso o CMD da imagem será sobrescrito pelo comando informado;
 - LABEL: Adiciona metadados a uma imagem, informações adicionais que servirão para identificar versão, tipo de licença, ou host, lembrando que a cada nova instrução LABEL é criada uma nova layer, o Docker recomenda que você não use muitas LABEL. É possível realizar filtragens posteriormente utilizando essas LABEL.
 - EXPOSE: Expõem uma ou mais portas, isso quer dizer que o container quando iniciado poderá ser acessível através dessas portas;
 - ENV: Instrução que cria e atribui um valor para uma variável dentro da imagem, isso é útil para realizar a instalação de alguma aplicação ou configurar um ambiente inteiro.
 - ADD: Adiciona arquivos locais  ou que estejam em uma url, para dentro da imagem.
 - COPY: Copia arquivos ou diretórios locais para dentro da imagem.
 - ENTRYPOINT: Informa qual comando será executado quando um container for iniciado utilizando esta imagem, diferentemente do CMD, o ENTRYPOINT não é sobrescrito, isso quer dizer que este comando será sempre executado.
 - VOLUME: Mapeia um diretório do host para ser acessível pelo container;
 - USER: Define com qual usuário serão executadas as instruções durante a geração da imagem;
 - WORKDIR: Define qual será o diretório de trabalho (lugar onde serão copiados os arquivos, e criadas novas pastas)

O DockerFile da nossa aplicação vai ficar mais ou menos assim:

```dockerfile
# Determinamos qual a imagem usaremos como base da nossa aplicação
# Link dockerhub https://hub.docker.com/_/python/ 
FROM python:3.7-slim

# Instalamos o pipenv no cotainer. Como foi dito anteriormente, temos que dizer o que precisa
# ser feito na nossa receita de bolo
RUN pip install pipenv

# Definimos o lugar onde o projeto ficará
WORKDIR /app

# Copiamos tudo que precisamos para o APP rodar
COPY ./src ./src/
COPY app.py .
COPY Pipfile .
COPY logging.yaml .

# Instalamos as dependências do python. Execução do pipfile.
RUN pipenv install --skip-lock --deploy --system

# Definimos as váriaveis do python
ENV PYTHONPATH=/usr/local/lib/python3.6/
ENV PYTHONPATH=/app

# Qual porta será exposta pelo nosso container
EXPOSE 5001

# Qual comando será executado na execução do nosso container, nesse caso, executamos a nossa aplicação
CMD [ "python", "app.py" ]
```

# Gerando o container e rodando a aplicação

Para subir o container vamos executar o comando no cmd:

```
docker build -t app-docker .
```

Que basicamente é: Docker, **construa** o container **app-docker** a partir desta pasta. 

O docker tem suporte a ajuda em todos os seus comandos, caso tenha dúvida escreva o comando e **--help**, no nosso caso: `docker build --help`.

A partir deste momento o container está pronto para uso, para subí-lo vamos utilizar o `docker run`.

```
docker run -p 5001:5001 --name app-docker -d app-docker
```

Se tudo deu certo, o docker irá mostrar o **ID** gerado para o container logo após o comando. Para ver o container devemos executar o comando `docker ps`, que exibirá:

```
CONTAINER ID        IMAGE                   COMMAND             CREATED             STATUS              PORTS                    NAMES
3abd9b8240b7        app-docker             "python app.py"     38 seconds ago      Up 35 seconds       0.0.0.0:5001->5001/tcp   app-docker
```

Pronto. Agora vamos acessar `localhost:5001` no navegador e vamos ter a resposta do nosso app de dentro do container. 

Tudo isso é muito legal, mas ainda precisamos disponibilizar o container para que outras pessoas tenham acesso. Nesse exemplo vou utilizar o próprio (DockerHub)[https://hub.docker.com].

# Publicando o container

A primeira coisa que precisamos fazer é gerar uma tag para o nosso container, que basicamente é a maneira de identificar as diferentes versões de um mesmo container. Para a tag precisamos informar a imagem e a definição da tag, vale a pena verificar o help `docker tag --help`.


```
# docker tag suaimagem suatag
docker tag app-docker cassioroos/app-docker
```
Agora vamos enviar o nosso container para o DockerHub.

```
docker push cassioroos/app-docker
```

Partiu ver como ficou. Vamos acessar o DockerHub e ver a nossa imagem publicada. Ela deve estar no repositorio. 

<img src="https://i.imgur.com/rHS2uBR.png" style="height:400px;"/>

Agora sempre que precisarmos subir nossa aplicação, podemos fazer isso em qualquer máquina, sem preocupação com ambiente e configurações.

```
docker pull cassioroos/app-docker
```

FIM.


Espero que tenham gostado.