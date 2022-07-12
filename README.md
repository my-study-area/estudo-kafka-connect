# estudo-kafka-connect
<p>
    <img alt="GitHub top language" src="https://img.shields.io/github/languages/top/my-study-area/estudo-kafka-connect">
    <a href="https://github.com/my-study-area">
        <img alt="Made by" src="https://img.shields.io/badge/made%20by-adriano%20avelino-gree">
    </a>
    <img alt="Repository size" src="https://img.shields.io/github/repo-size/my-study-area/estudo-kafka-connect">
    <a href="https://github.com/my-study-area/estudo-kafka-connect/commits/master">
    <img alt="GitHub last commit" src="https://img.shields.io/github/last-commit/my-study-area/estudo-kafka-connect">
    </a>
</p>

Projetos de estudo de kafka-connect
## Iniciando

### Twelve Days of SMT 
#### Twelve Days of SMT 🎄 - Day 10: ReplaceField
```bash
# cria um conector para criar dados automáticos no tópico source-voluble-datagen-day10-00
curl -i -X PUT -H  "Content-Type:application/json" \
    http://localhost:8083/connectors/source-voluble-datagen-day10-00/config \
    -d '{
        "connector.class"                              : "io.mdrogalis.voluble.VolubleSourceConnector",
        "genkp.day10-transactions.with"                : "#{Internet.uuid}",
        "genv.day10-transactions.cost.with"            : "#{Commerce.price}",
        "genv.day10-transactions.units.with"           : "#{number.number_between '\''1'\'','\''99'\''}",
        "genv.day10-transactions.txn_date.with"        : "#{date.past '\''10'\'','\''DAYS'\''}",
        "genv.day10-transactions.cc_num.with"          : "#{Business.creditCardNumber}",
        "genv.day10-transactions.cc_exp.with"          : "#{Business.creditCardExpiry}",
        "genv.day10-transactions.card_type.with"       : "#{Business.creditCardType}",
        "genv.day10-transactions.customer_remarks.with": "#{BackToTheFuture.quote}",
        "genv.day10-transactions.item.with"            : "#{Beer.name}",
        "topic.day10-transactions.throttle.ms"         : 1000
    }'

# consome as mensagem deserializando usando schema-registry e avro
docker-compose exec kafkacat sh -c "kafkacat -b broker:29092 -t day10-transactions \
-s value=avro -r http://schema-registry:8081 -C -J -e | jq [.payload]"

# cria conector para salvar dados no mysql usando blacklist
curl -i -X PUT -H "Accept:application/json" \
  -H  "Content-Type:application/json" http://localhost:8083/connectors/sink-jdbc-mysql-day10-01/config \
  -d '{
      "connector.class"            : "io.confluent.connect.jdbc.JdbcSinkConnector",
      "connection.url"             : "jdbc:mysql://mysql:3306/demo",
      "connection.user"            : "mysqluser",
      "connection.password"        : "mysqlpw",
      "topics"                     : "day10-transactions",
      "tasks.max"                  : "4",
      "auto.create"                : "true",
      "auto.evolve"                : "true",
      "transforms"                 : "dropCC",
      "transforms.dropCC.type"     : "org.apache.kafka.connect.transforms.ReplaceField$Value",
      "transforms.dropCC.blacklist": "cc_num,cc_exp,card_type"
      }'

# conecta no mysql e mostra a estrutura da tabela sem os campos do blacklist
docker exec -it mysql bash -c 'mysql -u root -p$MYSQL_ROOT_PASSWORD demo'
desc `day10-transactions`;

# cria conector para salvar dados no mysql usando whitelist
curl -X PUT http://localhost:8083/connectors/source-jdbc-mysql-day10-00/config \
  -H "Content-Type: application/json" -d '{
    "connector.class"                  : "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url"                   : "jdbc:mysql://mysql:3306/demo",
    "connection.user"                  : "mysqluser",
    "connection.password"              : "mysqlpw",
    "topic.prefix"                     : "day10-",
    "poll.interval.ms"                 : 10000,
    "tasks.max"                        : 1,
    "table.whitelist"                  : "production_data",
    "mode"                             : "bulk",
    "transforms"                       : "selectFields",
    "transforms.selectFields.type"     : "org.apache.kafka.connect.transforms.ReplaceField$Value",
    "transforms.selectFields.whitelist": "item,cost,units,txn_date"
  }'

# cria conector para salvar dados no mysql renomeando campos
curl -i -X PUT -H "Accept:application/json" \
  -H  "Content-Type:application/json" http://localhost:8083/connectors/sink-jdbc-mysql-day10-02/config \
  -d '{
      "connector.class"            : "io.confluent.connect.jdbc.JdbcSinkConnector",
      "connection.url"             : "jdbc:mysql://mysql:3306/demo",
      "connection.user"            : "mysqluser",
      "connection.password"        : "mysqlpw",
      "topics"                     : "day10-transactions",
      "tasks.max"                  : "4",
      "auto.create"                : "true",
      "auto.evolve"                : "true",
      "transforms"                 : "renameTS",
      "transforms.renameTS.type"   : "org.apache.kafka.connect.transforms.ReplaceField$Value",
      "transforms.renameTS.renames": "txn_date:transaction_timestamp"
      }'
```

> Para instalar os conectores manualmente basta realizar o download do conector e criar um Dockerfile copiando a pasta `lib` e o arquivo `manifest.json` no diretório `/usr/share/confluent-hub-components/<NOME-CONECTOR>` dentro do container

####  Twelve Days of SMT 🎄 - Day 1: InsertField (timestamp)
```bash
# inicia os containers
dockercompose up -d

# cria um conector para gerar dados automáticos no tópico customers e transacations
curl -i -X PUT -H  "Content-Type:application/json" \
    http://localhost:8083/connectors/source-voluble-datagen-00/config \
    -d '{
        "connector.class"                      : "io.mdrogalis.voluble.VolubleSourceConnector",
        "genkp.customers.with"                 : "#{Internet.uuid}",
        "genv.customers.name.with"             : "#{HitchhikersGuideToTheGalaxy.character}",
        "genv.customers.email.with"            : "#{Internet.emailAddress}",
        "genv.customers.location->city.with"   : "#{HitchhikersGuideToTheGalaxy.location}",
        "genv.customers.location->planet.with" : "#{HitchhikersGuideToTheGalaxy.planet}",
        "topic.customers.records.exactly"      : 10,

        "genkp.transactions.with"                : "#{Internet.uuid}",
        "genv.transactions.customer_id.matching" : "customers.key",
        "genv.transactions.cost.with"            : "#{Commerce.price}",
        "genv.transactions.card_type.with"       : "#{Business.creditCardType}",
        "genv.transactions.item.with"            : "#{Beer.name}",
        "topic.transactions.throttle.ms"         : 500
    }'

# lista os tópicos
docker-compose exec kafkacat sh -c "kafkacat -b broker:29092 -L | grep topic"

# consume as mensagens do tópico transactions criado no conector source-voluble-datagen-00
# obs: os valores do payload estarão serializados
docker-compose exec kafkacat sh -c "kafkacat -b broker:29092 -t transactions -C -J | jq"

# consome as mensagem deserializando usando schema-registry e avro
docker-compose exec kafkacat sh -c "kafkacat -b broker:29092 -t transactions \
-s value=avro -r http://schema-registry:8081 -C -J | jq"

# cria conector jdbc para alimentar o mysql
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/sink-jdbc-mysql-00/config \
    -d '{
          "connector.class"     : "io.confluent.connect.jdbc.JdbcSinkConnector",
          "connection.url"      : "jdbc:mysql://mysql:3306/demo",
          "connection.user"     : "mysqluser",
          "connection.password" : "mysqlpw",
          "topics"              : "transactions",
          "tasks.max"           : "4",
          "auto.create"         : "true"
        }'

# acessa o container mysql com o usuário root
# descreve a estrutura da tabela
docker exec -it mysql bash -c 'mysql -u root -p$MYSQL_ROOT_PASSWORD demo'
desc transactions;

// resposta
+-------------+------+------+-----+---------+-------+
| Field       | Type | Null | Key | Default | Extra |
+-------------+------+------+-----+---------+-------+
| customer_id | text | YES  |     | NULL    |       |
| cost        | text | YES  |     | NULL    |       |
| item        | text | YES  |     | NULL    |       |
| card_type   | text | YES  |     | NULL    |       |
+-------------+------+------+-----+---------+-------+

# atualiza conector adicionando coluna messageTS
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/sink-jdbc-mysql-00/config \
    -d '{
          "connector.class"     : "io.confluent.connect.jdbc.JdbcSinkConnector",
          "connection.url"      : "jdbc:mysql://mysql:3306/demo",
          "connection.user"     : "mysqluser",
          "connection.password" : "mysqlpw",
          "topics"              : "transactions",
          "tasks.max"           : "4",
          "auto.create"         : "true",
          "auto.evolve"         : "true",
          "transforms"                         : "insertTS",
          "transforms.insertTS.type"           : "org.apache.kafka.connect.transforms.InsertField$Value",
          "transforms.insertTS.timestamp.field": "messageTS"
        }'

# acessa o container mysql com o usuário root
# descreve a estrutura da tabela
docker exec -it mysql bash -c 'mysql -u root -p$MYSQL_ROOT_PASSWORD demo'
desc transactions;

//resposta
+-------------+-------------+------+-----+---------+-------+
| Field       | Type        | Null | Key | Default | Extra |
+-------------+-------------+------+-----+---------+-------+
| customer_id | text        | YES  |     | NULL    |       |
| cost        | text        | YES  |     | NULL    |       |
| item        | text        | YES  |     | NULL    |       |
| card_type   | text        | YES  |     | NULL    |       |
| messageTS   | datetime(3) | YES  |     | NULL    |       |
+-------------+-------------+------+-----+---------+-------+

# cria o bucket 
aws --endpoint-url=http://localhost:4566 s3 mb s3://rmoff-smt-demo-01

# lista os buckets criados
aws s3api list-buckets --query "Buckets[].Name" \
--endpoint-url=http://localhost:4566

# cria conector com s3
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/sink-s3-00/config \
    -d '{
          "connector.class"        : "io.confluent.connect.s3.S3SinkConnector",
          "storage.class"          : "io.confluent.connect.s3.storage.S3Storage",
          "store.url" : "http://localstack:4566",
          "s3.region"              : "us-west-2",
          "s3.bucket.name"         : "rmoff-smt-demo-01",
          "topics"                 : "customers,transactions",
          "tasks.max"              : "4",
          "flush.size"             : "16",
          "format.class"           : "io.confluent.connect.s3.format.json.JsonFormat",
          "schema.generator.class" : "io.confluent.connect.storage.hive.schema.DefaultSchemaGenerator",
          "schema.compatibility"   : "NONE",
          "partitioner.class"      : "io.confluent.connect.storage.partitioner.DefaultPartitioner"
        }'

# lista todos os arquivo no s3
aws --endpoint-url=http://localhost:4566 s3 ls s3://rmoff-smt-demo-01 \
--recursive --human-readable --summarize

# lista todos os arquivos num bucket s3
aws --endpoint-url=http://localhost:4566 s3api list-objects --bucket rmoff-smt-demo-01 \
--output text --query "Contents[].{Key: Key}"
```
Para acessar as mensagens criadas pelo conector `source-voluble-datagen-00` (io.mdrogalis.voluble.VolubleSourceConnector), acesse [http://localhost:9021/](http://localhost:9021/) > Menu lateral `Topics` > click no tópico `transactions` e click em `Messages`
### Curso: Kafka Connect 101 (Confluent)
```bash
# inicia container
docker-compose up -d

# cria connector datagen
curl -i -X PUT http://localhost:8083/connectors/datagen_local_01/config \
     -H "Content-Type: application/json" \
     -d '{
            "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
            "key.converter": "org.apache.kafka.connect.storage.StringConverter",
            "kafka.topic": "pageviews",
            "quickstart": "pageviews",
            "max.interval": 1000,
            "iterations": 10000000,
            "tasks.max": "1"
        }'

# verifica se o conector está executando
curl -s http://localhost:8083/connectors/datagen_local_01/status

# lê as mensagens do tópico pageviews
docker-compose exec connect kafka-avro-console-consumer \
 --bootstrap-server broker:9092 \
 --property schema.registry.url=http://schema-registry:8081 \
 --topic pageviews \
 --property print.key=true \
 --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
 --property key.separator=" : " \
 --max-messages 10

 # verifica o stack trace utilizando curl e jq
 curl -s "http://localhost:8083/connectors/source-debezium-orders-00/status"
| jq '.tasks[0].trace'

# logs do serviço connect via docker-compose
docker-compose logs -f connect

# exemplo de alteração do level de log sem reiniciar conector
curl -s -X PUT -H "Content-Type:application/json" \
    http://localhost:8083/admin/loggers/io.debezium \
    -d '{"level": "TRACE"}'

# verifica o status de um conector e filtra a saída
# exibindo o nome e o status usando jq
curl -v http://localhost:8083/connectors/transform2/status | \
jq -c '[.name, .tasks[].state]'

# exemplo de conector com dead letter queue
# gravando o motivo do erro, no header da mensagem
curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
        "name": "file_sink_03",
        "config": {
                "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
                "topics":"test_topic_json",
                "value.converter":"org.apache.kafka.connect.json.JsonConverter",
                "value.converter.schemas.enable": false,
                "key.converter":"org.apache.kafka.connect.json.JsonConverter",
                "key.converter.schemas.enable": false,
                "file": "/data/file_sink_03.txt",
                "errors.tolerance": "all",
                "errors.deadletterqueue.topic.name":"dlq_file_sink_03",
                "errors.deadletterqueue.topic.replication.factor": 1,
                "errors.deadletterqueue.context.headers.enable":true
                }
        }'

# exemplo de conector, exibindo o motivo do erro no log do worker
# "errors.log.enable":true -> habilita o log
# "errors.log.include.messages":true -> habilita metadata no log
#
curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
        "name": "file_sink_04",
        "config": {
                "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
                "topics":"test_topic_json",
                "value.converter":"org.apache.kafka.connect.json.JsonConverter",
                "value.converter.schemas.enable": false,
                "key.converter":"org.apache.kafka.connect.json.JsonConverter",
                "key.converter.schemas.enable": false,
                "file": "/data/file_sink_04.txt",
                "errors.tolerance": "all",
                "errors.log.enable":true,
                "errors.log.include.messages":true
                }
        }'

# verifica a versão e id do cluster
curl -v http://localhost:8083

# lista os plugins de conectores instalados
curl -v http://localhost:8083/connector-plugins

# lista os conectores
curl -v http://localhost:8083/connectors/

# lista as configurações do conector
curl -v http://localhost:8083/connectors/file_sink_04/config 
```


Fonte: [Kafka Connect 101](https://developer.confluent.io/learn-kafka/kafka-connect/intro/)

### Introduction to Kafka Connectors
```bash
# entra no diretório
cd baeldung

# inicia os containers
docker-compose up -d

# acessa container do kafka-connect
docker-compose run --rm kafka-connect bash

# adiciona arquivo para o source connector
echo -e "foo\nbar\n" > ./test.txt

# inicia worker
connect-standalone \
  ./connect-standalone.properties \
  ./connect-file-source.properties \
  ./connect-file-sink.properties

# inicia outro terminal do bash do kafka-connect
docker-compose run --rm kafka-connect bash

# consome as mensagens lançadas pelo worker
kafka-console-consumer --bootstrap-server kafka:9092 \
--topic connect-test --from-beginning

# resposta esperada
# {"schema":{"type":"string","optional":false},"payload":"foo"}
# {"schema":{"type":"string","optional":false},"payload":"bar"}

# inicia o kafka-connect no modo distribuído
connect-distributed connect-distributed.properties

# adiciona um connector source
curl -d @connect-file-source.json \
  -H "Content-Type: application/json" \
  -X POST http://localhost:8083/connectors -v

# adiciona um connector sink via api
curl -d @connect-file-sink.json \
  -H "Content-Type: application/json" \
  -X POST http://localhost:8083/connectors

# consome as mensagens do tópico connect-distributed
kafka-console-consumer --bootstrap-server kafka:9092 \
--topic connect-distributed --from-beginning

# deleta os connectors via api
curl -X DELETE http://localhost:8083/connectors/local-file-source
curl -X DELETE http://localhost:8083/connectors/local-file-sink

# inicia kafka connect com configuração para transformar dados
connect-distributed connect-distributed-transformer.properties

# adiciona um connector source com transformação de dados
curl -v -d @data/connect-file-source-transform.json \
  -H "Content-Type: application/json" \
  -X POST http://localhost:8083/connectors

# cria consumidor para o tópico connect-transformation
kafka-console-consumer --bootstrap-server kafka:9092 \
--topic connect-transformation --from-beginning

#  acessa container do kafka-connect
# instala conector do confluent-hub 
docker-compose run --rm kafka-connect bash
confluent-hub install confluentinc/kafka-connect-mqtt:1.0.0-preview
```

connect-file-source.json:
```json
{
    "name": "local-file-source",
    "config": {
        "connector.class": "FileStreamSource",
        "tasks.max": 1,
        "file": "test-distributed.txt",
        "topic": "connect-distributed"
    }
}
```

connect-file-sink.json:
```json
{
    "name": "local-file-sink",
    "config": {
        "connector.class": "FileStreamSink",
        "tasks.max": 1,
        "file": "test-distributed.sink.txt",
        "topics": "connect-distributed"
    }
}
```

fonte: [Introduction to Kafka Connectors](https://www.baeldung.com/kafka-connectors-guide)

### Projeto Kafka Connect: Integração entre sistemas (MySQL /Elasticsearch)
```bash
# entra no diretório
cd full-cycle

# inicia os containers
docker-compose up -d

# Em caso de erro no elastic search
# [1]: max virtual memory areas vm.max_map_count [65530] is too low, 
# increase to at least [262144]
sudo echo "vm.max_map_count=262144" >>  /etc/sysctl.conf 
sudo sysctl -w vm.max_map_count=262144 # vm.max_map_count=262144

# acessa banco de dados products no mysql
mysql -u root -p products -h 127.0.0.1 -P 33600

# cria a tabela product
create table products(id int, name varchar(255));

# insere registro na tabela products
insert into products values(1, "carro");

# seleciona os registros da tabela products com 
# visualização por linha
select * from products order by id desc limit 10\G;
*************************** 1. row ***************************
  id: 2
name: bicicleta
*************************** 2. row ***************************
  id: 1
name: carro
2 rows in set (0.00 sec)
```

Para utilizar o control-center no navegador acesse [http://localhost:9021/](http://localhost:9021/)
- acesse [http://localhost:9021/clusters/CqQIIf5IRYCCBCjzynuqSA/management/connect/connect-default/connectors](http://localhost:9021/clusters/CqQIIf5IRYCCBCjzynuqSA/management/connect/connect-default/connectors) para adicionar os conectores para o mysql (`full-cycle/mysql.properties`) e elastic search (`full-cycle/es-skink.properties`).

Para utilizar o Kibana acesse [http://localhost:5601/](http://localhost:5601/)
- acesse [http://localhost:5601/app/management/kibana/indexPatterns](http://localhost:5601/app/management/kibana/indexPatterns) para cria o índice `mysql-server*`
- acesse [http://localhost:5601/app/discover#/](http://localhost:5601/app/discover#/) para visualizar os dados. Obs: no menu lateral esquerdo, em `Available fields`, selecione os campos: **payload.before.id** e	**payload.before.nome** para melhorar a visualização dos dados.

fonte: [Kafka Connect: Integração entre sistemas (MySQL /Elasticsearch)](https://www.youtube.com/watch?v=qO4JL38_F1s&ab_channel=FullCycle)

## Api

curl -v http://localhost:8083/connector-plugins

## Links
- [Conceitos de kafka Connect](https://docs.confluent.io/platform/current/connect/concepts.html)
- [Iniciando com Kafka Connect](https://docs.confluent.io/home/connect/self-managed/userguide.html)
- [Curso de Kafka Connect](https://developer.confluent.io/learn-kafka/kafka-connect/intro/)
- [Apache Kafka Connect](https://kafka.apache.org/documentation/#connect)
- [Kafka Connect Transformations](https://docs.confluent.io/platform/current/connect/transforms/overview.html)
- [Confluent Hub](https://www.confluent.io/hub/)
- [Kafka Connect with FileStreamSource](https://docs.confluent.io/platform/7.0.1/connect/quickstart.html)
- [Kafka Connect Converter](https://rmoff.net/2019/05/08/when-a-kafka-connect-converter-is-not-a-_converter_/)
- [Dados Mockados com Kafka Connect Datagen](https://developer.confluent.io/tutorials/kafka-connect-datagen/kafka.html)
- [Running Confluent Kafka Connect Datagen Plugin Quickstart Template Locally with Docker](https://thecodinginterface.com/blog/kafka-connect-datagen-plugin/)
- [Quickstart parameters in kafka-connect-datagen](https://github.com/confluentinc/kafka-connect-datagen/blob/master/README.md#kafka-connect-datagen-specific-parameters)
- [Demo examples of Kafka Connect](https://github.com/confluentinc/demo-scene/#kafka-connect)
- [kcat (formerly kafkacat) Utility](https://docs.confluent.io/platform/current/app-development/kafkacat-usage.html#metadata-listing-mode)
- [Conceitos de Dead Letter Queue no Kafka Connect](https://docs.confluent.io/platform/current/connect/concepts.html#dead-letter-queues)
- [Kafka Connect Deep Dive – Error Handling and Dead Letter Queues](https://www.confluent.io/blog/kafka-connect-deep-dive-error-handling-dead-letter-queues/?_ga=2.82422291.464758877.1647995300-1077278967.1647804227)
- [Connect REST Interface](https://docs.confluent.io/platform/current/connect/references/restapi.html)
- [Changing the Logging Level for Kafka Connect Dynamically](https://rmoff.net/2020/01/16/changing-the-logging-level-for-kafka-connect-dynamically/)
- [Monitoring Kafka Connect and Connectors](https://docs.confluent.io/platform/current/connect/monitoring.html)
- [Amazon S3 Sink Connector Configuration Properties](https://docs.confluent.io/kafka-connect-s3-sink/current/configuration_options.html#storage)
- [List all Files in an S3 Bucket with AWS CLI](https://bobbyhadz.com/blog/aws-cli-list-all-files-in-bucket)
- [Connector Developer Guide](https://docs.confluent.io/platform/current/connect/devguide.html)
- [How to Write a Connector for Kafka Connect – Deep Dive into Configuration Handling](https://www.confluent.io/blog/write-a-kafka-connect-connector-with-configuration-handling/?_ga=2.91818522.1207622556.1655507562-1631858859.1655396982)
- [4 Steps to Creating Apache Kafka Connectors with the Kafka Connect API](https://www.confluent.io/blog/create-dynamic-kafka-connect-source-connectors/?_ga=2.91818522.1207622556.1655507562-1631858859.1655396982)
- [Create a test data generator using Kafka Connect](https://zendesk.engineering/create-a-test-data-generator-using-kafka-connect-f0a2419af76a)
  - [Kafka Connect Datagen Connector](https://github.com/xushiyan/kafka-connect-datagen)
- [Kafka Connect Sample Connector](https://github.com/riferrei/kafka-source-connector)
  - [Building your First Connector for Kafka Connect](https://www.youtube.com/watch?v=EXviLqXFoQI)
