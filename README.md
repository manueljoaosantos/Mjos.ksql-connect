# Mjos.ksql-connect
## Running Locally
We're deploying the following components with Docker compose:

- Zookeeper
- Kafka
- ksqlDB server (With Kafka Connect)
- ksqlDB CLI
- MySQL

```sh
docker-compose up &
```
```sh
docker exec -it ksqldb-cli ksql http://ksqldb-server:8088
```
```sql
SET 'auto.offset.reset' = 'earliest';
```
```sql
CREATE
SOURCE CONNECTOR mysql_source_connector
WITH (
  'connector.class' = 'io.confluent.connect.jdbc.JdbcSourceConnector',
  'connection.url' = 'jdbc:mysql://mysql:3306/football',
  'connection.user' = 'root',
  'connection.password' = 'root',
  'table.whitelist' = 'players',
  'mode' = 'incrementing',
  'incrementing.column.name' = 'id',
  'topic.prefix' = '',
  'key'='id',
  'key.converter'='org.apache.kafka.connect.storage.StringConverter',
  'value.converter'='org.apache.kafka.connect.json.JsonConverter',
  'value.converter.schemas.enable' = false
);
```
```sql
SHOW CONNECTORS;
```
```sql
SHOW TOPICS;
```
```sql
CREATE TABLE players
(
    id          VARCHAR PRIMARY KEY,
    name        VARCHAR(50),
    team        VARCHAR(50),
    nationality VARCHAR(50)
)
    WITH (
        KAFKA_TOPIC = 'players',
        VALUE_FORMAT = 'JSON',
        PARTITIONS = 1
        );
```
```sql
SELECT * FROM players EMIT CHANGES;
```

```sql
SELECT name, team FROM players EMIT CHANGES;
```

```sql
SELECT name, UCASE(team) team FROM players EMIT CHANGES;
```
```sql
SELECT name, 
       team,
       CASE
           WHEN name = 'Lionel Messi' THEN 'GOAT'
           ELSE 'PLAYER'
       END status
       FROM players
       EMIT CHANGES;
```
```sql
SELECT * FROM players
WHERE team = 'Manchester City'
EMIT CHANGES;
```
```sql
SELECT * FROM players
WHERE team = 'Manchester City'
AND nationality = 'Belgian'
EMIT CHANGES;
```
```sql
CREATE
STREAM match_event (
    id VARCHAR KEY,
    event_type VARCHAR,
    player_id VARCHAR,
    home boolean
) WITH (
    KAFKA_TOPIC='match_event',
    VALUE_FORMAT='JSON',
    PARTITIONS=1
);
```
```sql
SELECT * FROM match_event EMIT CHANGES;
```
```sql
INSERT INTO match_event
VALUES ('1', 'GOAL', '1', true);
```
```sql
SELECT * FROM match_event EMIT CHANGES;
```
```sql
INSERT INTO match_event
VALUES ('1', 'ASSIST', '1', true);
```

```sql
SELECT * FROM match_event
WHERE event_type = 'ASSIST'
EMIT CHANGES;
```
```sql
SELECT
    id,
    COUNT(id) home_goals
FROM match_event
WHERE home AND event_type = 'GOAL'
GROUP BY id
EMIT CHANGES;
```
```sql
INSERT INTO match_event
VALUES ('1', 'GOAL', '1', true);
INSERT INTO match_event
VALUES ('1', 'GOAL', '2', false);
```
```sql
SELECT
    id,
    COUNT(id) away_goals
FROM match_event
WHERE NOT home AND event_type = 'GOAL'
GROUP BY id
EMIT CHANGES;
```
```sql
INSERT INTO match_event
VALUES ('2', 'GOAL', '1', true);
INSERT INTO match_event
VALUES ('2', 'ASSIST', '2', false);
INSERT INTO match_event
VALUES ('2', 'GOAL', '2', false);
```
```sql
SELECT id,
       sum(
               CASE
                   WHEN home AND event_type = 'GOAL' THEN 1
                   ELSE 0
                   END
           ) home_goals,
       sum(
               CASE
                   WHEN NOT home AND event_type = 'GOAL' THEN 1
                   ELSE 0
                   END
           ) away_goals
FROM match_event
GROUP BY id 
EMIT CHANGES;
```
```sql
CREATE TABLE match_results 
WITH (
    KAFKA_TOPIC='match_results',
    VALUE_FORMAT='JSON'
    ) AS
SELECT id, 
       sum(
               CASE
                   WHEN home AND event_type = 'GOAL' THEN 1
                   ELSE 0
                   END
           ) home_goals,
       sum(
               CASE
                   WHEN NOT home AND event_type = 'GOAL' THEN 1
                   ELSE 0
                   END
           ) away_goals
FROM match_event
GROUP BY id;
```
```sql
SELECT * FROM match_results EMIT CHANGES;
```
```shell
 docker exec kafka kafka-console-consumer --bootstrap-server localhost:9092 --from-beginning --topic match_results
```
```shell
docker exec kafka kafka-console-consumer --bootstrap-server localhost:9092 --from-beginning --topic match_results --property print.key=true --property key.separator=":" 
```

```sql
SELECT p.id, p.name, count(me.id) goals
FROM match_event me
JOIN players p on me.player_id = p.id
WHERE me.event_type = 'GOAL'
GROUP BY p.id, p.name
EMIT CHANGES;
```

```sql
SELECT p.id AS player_id,
       p.name AS name,
       p.nationality AS nationality,
       SUM(
               CASE
                   WHEN me.event_type = 'GOAL' THEN 1
                   ELSE 0
                   END
           ) goals,
       CAST(SUM(
               CASE
                   WHEN me.event_type = 'GOAL' THEN 1
                   ELSE 0
                   END
           )
           AS DOUBLE) / cast(COUNT_DISTINCT((me.id)) AS DOUBLE) avg_goals,
       SUM(
               CASE
                   WHEN me.event_type = 'ASSIST' THEN 1
                   ELSE 0
                   END
           ) assists
FROM match_event me
         JOIN players p
              ON p.id = me.player_id
GROUP BY p.id, p.name, p.nationality
EMIT CHANGES;
```
```sql
CREATE TABLE player_stats
WITH (
    KAFKA_TOPIC='player_stats',
    FORMAT='JSON',
    PARTITIONS=1
    ) AS
SELECT p.id AS player_id,
       p.name AS name,
       p.nationality AS nationality,
       SUM(
               CASE
                   WHEN me.event_type = 'GOAL' THEN 1
                   ELSE 0
                   END
           ) goals,
       CAST(SUM(
               CASE
                   WHEN me.event_type = 'GOAL' THEN 1
                   ELSE 0
                   END
           )
           AS DOUBLE) / cast(COUNT_DISTINCT((me.id)) AS DOUBLE) avg_goals,
       SUM(
               CASE
                   WHEN me.event_type = 'ASSIST' THEN 1
                   ELSE 0
                   END
           ) assists
FROM match_event me
         JOIN players p
              ON p.id = me.player_id
GROUP BY p.id, p.name, p.nationality;
```
```shell
docker exec kafka kafka-console-consumer --bootstrap-server localhost:9092 --from-beginning --topic player_stats --property print.key=true --property key.separator=":"
```