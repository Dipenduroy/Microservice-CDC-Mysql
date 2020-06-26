# Microservice-CDC-Mysql

- [Microservice-CDC-Mysql Tutorial using Debezium Source and Confluent Sink Connector](#microservice-cdc-mysql)
  * [Start tutorial](#start-tutorial-demo)
    + [List - Register Connectors](#list-connectors)
      - [Source Connector](#register-source-connector)
      - [Sink Connector](#register-sink-connector)
    + [Verify CDC Replication to destination database](#using-mysql-and-apicurio-registry)
      - [Connect to mysql source](#connect-to-mysql-source)
      - [Verify records in source](#verify-records-in-source)
      - [Connect to mysql destination](#connect-to-mysql-destination)
      - [Verify records in destination](#verify-records-in-destination)
      - [Insert records in source](#insert-records-in-source)
      - [Verify whether changes recorded in destination](#verify-records-in-destination)
  * [End tutorial](#end-the-demo)
  * [References](#references)

### Start tutorial demo
`docker-compose -f docker-compose-mysql.yaml up -d`

### Check connector plugins list
localhost:8083/connector-plugins

### List connectors
localhost:8083/connectors/

### Register source connector
`curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-mysql.json`

### Consume messages from a Debezium topic
`docker-compose -f docker-compose-mysql.yaml exec kafka /kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server kafka:9092 \
    --from-beginning \
    --property print.key=true \
    --topic dbserver1.inventory.customers`

### Register sink connector
`curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-mysql-sink.json`

Register multiple sink connector for different tables, if primary key or the configuration differs

### To check status of a connector: replace name of the connector in "jdbc-sink-connector"
http://localhost:8083/connectors/jdbc-sink-connector/status

### To delete a connector: replace name of the connector in "jdbc-sink-connector"
`curl -X DELETE localhost:8083/connectors/jdbc-sink-connector`

### To restart a connector
`curl -X POST localhost:8083/connectors/jdbc-sink-connector/restart`

### Connect to Mysql source:
`docker-compose -f docker-compose-mysql.yaml exec mysql bash -c 'mysql -u $MYSQL_USER -p$MYSQL_PASSWORD inventory'`

### Connect to Mysql Destination:
`docker-compose -f docker-compose-mysql.yaml exec mysqldestination bash -c 'mysql -u $MYSQL_USER -p$MYSQL_PASSWORD inventory'`

### Verify records in source
`select * from customers;`

### Update records in source
`update customers set first_name='dip';`

### Insert records in source
`insert into customers(id,first_name,last_name,email) values(1005,'1','1','1');`

### Verify records in destination
`select * from kafka_customers;`

### Delete a record in source
`delete from customers where id=1005;` 

### Add column in source : Column will only be added if any new insert sql statement is found with the new column
`ALTER TABLE customers
  ADD phone varchar(40)  NULL
    AFTER email;`
    
### Rename column name in source : Column will only be added if any new insert sql statement is found with the new column,but old data in not available in new column. So only rename columns if data are not added in old field
`ALTER TABLE customers
  CHANGE COLUMN phone mobile
    varchar(20) NULL;`
    
### Remove a column in source : Column is not removed : Remove is not yet supported
`ALTER TABLE customers
  DROP COLUMN mobile;`    
  
### Index/Unique index/Auto increment column/Foreign Keys is not added in destination, but due to it no adverse effect is found yet

### Check whether log bin is enabled in mysql
`SHOW VARIABLES LIKE 'log_bin';`

### Add multiple comma seperated tables name that needs to be replicated should be placed in topics key of sink connector configuration 
"topics": "customers,addresses,products"

### Add prefix to the destination tables

Eg: For customers table, kafka_customers will be created in the destination.

"table.name.format":"kafka_${topic}",

### Add multiple parallel workers for the tasks
The maximum number of tasks that should be created for this connector. The connector may create fewer tasks if it cannot achieve this level of parallelism.
Eg:
"tasks.max" : 3

### Optional : Get mysql Server id to be placed in : database.server.id
`SELECT @@server_id`

Refer [debezium config](https://debezium.io/documentation/reference/1.1/connectors/mysql.html#mysql-property-database-server-id:~:text=A%20numeric%20ID%20of%20this%20database,we%20recommend%20setting%20an%20explicit%20value.) for more

### End the demo
`docker-compose -f docker-compose-mysql.yaml down`

### References
Thanks to the below contributors

1. [Microservices](https://microservices.io/patterns/data/database-per-service.html) - Database per service architecture
2. Use [CQRS](https://microservices.io/patterns/data/cqrs.html) to overcome Database per service pitfalls
3. Inspired by [Eventuate Tram](https://eventuate.io/abouteventuatetram.html)
4. Debezium Mysql Source [Connector](https://debezium.io/documentation/reference/1.1/connectors/mysql.html)
5. Confluent JDBC Sink Connector [configuration](https://docs.confluent.io/current/connect/kafka-connect-jdbc/sink-connector/sink_config_options.html#sink-config-options)
6. Manage [Confluent Kafka connector](https://docs.confluent.io/3.2.0/connect/managing.html)
7. Inplace of Debezium, [Confluent JDBC Source Connector configuration](https://docs.confluent.io/current/connect/kafka-connect-jdbc/source-connector/source_config_options.html#jdbc-source-configs) can also be used.
8. Other Debezium source connector [examples](https://github.com/debezium/debezium-examples)