# Mjos.ksql-connect
## Running Locally
We're deploying the following components with Docker compose:

- Zookeeper
- Kafka
- ksqlDB server (With Kafka Connect)
- ksqlDB CLI
- Schema Registry (To keep the schema of the data)
- MySQL
- Redis


```sh
docker-compose exec ksqldb-cli ksql http://ksqldb-server:8088
```
```sql
show connectors;
```
```sql
CREATE SOURCE CONNECTOR mysql_source_connector
WITH (
  'connector.class' = 'io.confluent.connect.jdbc.JdbcSourceConnector',
  'connection.url' = 'jdbc:mysql://mysql:3306/football',
  'connection.user' = 'root',
  'connection.password' = 'root',
  'table.whitelist' = 'players',
  'mode' = 'incrementing',
  'incrementing.column.name' = 'id',
  'topic.prefix' = '',
  'key'='id'
);
```
```sql
show connectors;
```
```sql
SHOW TOPICS;
```
```sql
CREATE SINK CONNECTOR redis_sink WITH (
  'connector.class'='com.github.jcustenborder.kafka.connect.redis.RedisSinkConnector',
  'tasks.max'='1',
  'topics'='players',
  'redis.hosts'='redis:6379',
  'key.converter'='org.apache.kafka.connect.converters.ByteArrayConverter',
  'value.converter'='org.apache.kafka.connect.converters.ByteArrayConverter'
);
```
```shell
docker-compose exec redis redis-cli
```
```sh
SELECT 1
```
```sh
GET 1
```
```
"\x00\x00\x00\x00\x01\x02\x18Lionel Messi\x12Paris Saint-Germain\x16Argentinian"
```