# estudo-kafka-connect

Projetos de estudo de kafka-connect
## Iniciando
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
```

Para utilizar o control-center no navegador acesse [http://localhost:9021/](http://localhost:9021/)
- acesse [http://localhost:9021/clusters/CqQIIf5IRYCCBCjzynuqSA/management/connect/connect-default/connectors](http://localhost:9021/clusters/CqQIIf5IRYCCBCjzynuqSA/management/connect/connect-default/connectors) para adicionar os conectores para o mysql (`full-cycle/mysql.properties`) e elastic search (`full-cycle/es-skink.properties`).

Para utilizar o Kibana acesse [http://localhost:5601/](http://localhost:5601/)
- acesse [http://localhost:5601/app/management/kibana/indexPatterns](http://localhost:5601/app/management/kibana/indexPatterns) para cria o índice `mysql-server*`
- acesse [http://localhost:5601/app/discover#/](http://localhost:5601/app/discover#/) para visualizar os dados. Obs: no menu lateral esquerdo, em `Available fields`, selecione os campos: **payload.before.id** e	**payload.before.nome** para melhorar a visualização dos dados.

fonte: [Kafka Connect: Integração entre sistemas (MySQL /Elasticsearch)](https://www.youtube.com/watch?v=qO4JL38_F1s&ab_channel=FullCycle)
