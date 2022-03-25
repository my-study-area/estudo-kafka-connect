# estudo-kafka-connect

Projetos de estudo de kafka-connect
## Iniciando

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
