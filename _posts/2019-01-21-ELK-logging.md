---
layout: post
title: "ELK - Elastisearh logstash kibana"
categories:
  - pt-br
tags:
  - pt-br
  - python
  - logging
featured-img: elk
last_modified_at: 2019-01-22T21:26:32-05:00
---

## Agora sim, um post sobre desenvolvimento

Nesse post quero mostrar como utilizar o [Elasticsearch, Logstash e Kibana](https://github.com/CassioRoos/logstash) para logs assincronos do python. Tenho um [projeto base](https://github.com/CassioRoos/python-base-project) que usarei para demontrar o funcionamento.

#### O que usaremos
[Docker](https://www.docker.com/get-started) para subir o ambiente ELK.
[Elasticsearch](https://www.elastic.co/products/elasticsearch) indexa e armazena.
[Logstash](https://www.elastic.co/products/logstash) receberá os logs do python.
[Kibana](https://www.elastic.co/products/kibana) visualização do que foi enviado.

## O porquê

Há alguns dias tive a necessidade de que os logs da minha aplicação python fossem descentralizados e que pudessem ser analizados de forma analítica. Foi então que comecei a pesquisar sobre logs no python e me deparei com muita coisa, inclusive em russo *quem nunca*, mas para fazer tudo funcionar como esperado levou algum tempo e muito teste. Foi então que resolvi fazer desse o primeiro tema do post. Um compilado com tudo que utilizei e de como configurar toda esta parafernalha.

## Subindo o ELK
> [O ELK é a junção das ferramentas Elasticsearch, Logstash e Kibana](https://www.elastic.co/elk-stack)

Para subir o ambiente executaremos o comando `docker-compose -f docker-compose-log.yml up -d`, que subirá o ambiente e deixará tudo disponível para acesso. Para envio do log, a porta será a [5000](localhost:5000) e a de acesso do Kibana a [5601](localhost:5601).

Para verificar se tudo deu certo, execute o comando `docker-compose -f docker-compose-log.yml ps`

```
elastic-painel    /docker-entrypoint.sh elas ...   Up      0.0.0.0:9200->9200/tcp, 0.0.0.0:9300->9300/tcp
kibana-painel     /docker-entrypoint.sh kibana     Up      0.0.0.0:5601->5601/tcp
logstash-painel   /usr/local/bin/docker-entr ...   Up      0.0.0.0:6000->5000/tcp, 5044/tcp, 0.0.0.0:7000->7000/tcp, 0.0.0.0:9600->9600/tcp
```

Antes de acessar o Kibana, precisamos dos logs sendo enviados para ele. Dessa maneira, ele saberá quais campos devem ser indexados.

## Subindo a aplicação base

Para subir o python eu prefiro utilizar o pipenv, ele faz a virtualizaçã do ambiente e deixa tudo mais fácil para trabalhar com mais projetos. Depois de clonar o [projeto base](https://github.com/CassioRoos/python-base-project), execute os comandos abaixo:

1. Caso não tenha o pipenv `pip install pipenv`.
2. Instalar as dependências do projeto `pipenv install` _necessário estar dentro da pasta do projeto_.
3. Para rodar o projeto `pipenv run app.py`, será exibida a porta onde a nossa aplicação está rodando.

```
Servidor rodando em http://LOCALHOST:5001. Pressione Ctrl + C para parar.
```

## Funcionamento
Tudo configurado e pronto para uso, mas vamos entender como as coisas trabalham e que impacto tem na aplicação. Começando com o arquivo de configuração do logs e a maneira que vamos lê-lo no python. 

- Só para deixar claro, a biblioteca que estamos utilizando do logstash para o paython é a [python-logstash-async](https://github.com/eht16/python-logstash-async)

```python
  with open('logging.yaml', 'r') as f:
      log_cfg = yaml.safe_load(f.read())
      logging.config.dictConfig(log_cfg)
```
Assim, simplesmente, estamos carregando o arquivo de configuração e passando para a aplicação. O que eu precisava descobrir é **como fazer com que todos os logs que importar no python tenham a mesma configuração**. 

```python
def log_request(handler):
    if handler.get_status() < 400:
        log_method = logging.info
    elif handler.get_status() < 500:
        log_method = logging.warning
    else:
        log_method = logging.error

    request_time = 1000.0 * handler.request.request_time()
    usuario = ''
    if hasattr(handler, 'usuario'):
        usuario = handler.usuario
    extradict = {
        "usuario": usuario,
        "metodo": handler.request.method,
        "caminho": handler.request.path,
        "status": handler.get_status(),
        "ip_remoto": handler.request.remote_ip,
        "uri": handler.request.uri,
        "tempo_request": request_time}

    camposlog = "{usuario:<10} {metodo:<8} {caminho:<20} {status:<4} {ip_remoto:<17}" \
                " {uri} {tempo_request}"
    log_method(camposlog.format(**extradict), extra=extradict)
```
Como precisava que o log se comportasse de uma maneira diferente da existe, sobrescrevi o mêtodo **log_request** e fiz a alteração necessária. Também precisava que independente do lugar que o log fosse gerado o sistema enviasse o log para o logstash, dessa maneira, alterei o root do log do pytho pelo yaml, assim qualquer log passara pela nossa rotina. 

Segue a configuração do yaml


```yaml
version: 1
disable_existing_loggers: False
formatters:
  logline:
    format: '%(asctime)s %(levelname)+8s %(message)s'
    datefmt: '%d/%m/%Y %H:%M:%S'
handlers:
  console:
    class: logging.StreamHandler
    formatter: logline
    level: INFO
  logstash:
    level: INFO
    class: logstash_async.handler.AsynchronousLogstashHandler
    host: localhost
    port: 6000
    database_path: logstash.db
root:
  handlers: 
    - console
    - logstash
  level: INFO
loggers:
  '':  # root logger
    level: INFO
    handlers:
      - console
      - logstash
    propagate: False
```

O arquivo se divide basicamente em 4 pedaços: *formatters, handlers, root e loggers*. Vamos focar nos handlers. O que fiz foi configurar 2 handlers para utilização, um *console* que tem basicamente a mesma função do já existente no python e logstash que vai fazer uso das ferramentas que configuramos.

O logstash tem 3 configurações obrigatórias para funcionamento:

1. Host: Endereço onde está o logstash.
2. Port: A porta de comunicação.
3. Database_path: O caminho onde o sistema salvará o logstash.db.

Outra informação que vale ressaltar é que o log do python gera tudo em um campo, **MESSAGE**. Para facilitar a leitura no kibana, colocar essas informações em um dicionário e passar no **EXTRA** do log_request.

## Utilização

Agora com o python rodando, acessar `localhost:5001` no browser. Se tudo deu certo, teremos a seguinte resposta:

```json
{"status": 200, "mensagem": "HELLO WORLD!"}
```

No prompt do python 

´´´
22/01/2019 22:27:56     INFO            GET      /                    200  ::1               / 7032.827615737915
´´´

Acessando `localhost:5601`, para visualizarmos o log no kibana, a tela de configuração irá aparecer com o botão create habilitado, conforme imagem a baixo:

<img src="https://i.imgur.com/bYMHGHw.png" style="height:300px;"/>

Depois de clicar em create, os índices serão exibidos e é só clicar em discover para ver os seus logs no kibana. Tudo pronto, os logs estão disponíveis, a partir de agora é configurar o kibana conforme a sua necessidade.

ESPERO TER AJUDADO.

Qualquer dúvida, sugestões e principalmente correções, deixe um comentário ou me envie um e-mail.