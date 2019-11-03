# Migration2Scylla
Create a Cassandra, intermediate and Scylla nodes using the docker-compose.yaml description file.
Start the containers by using
`docker-compose up`

First we load sample data into the Cassandra cluster.
Connect to the Cassandra node container using:
`docker exec -it cassandra-seed-node sh`
And execute : cqlsh

```
cqlsh:CREATE KEYSPACE dataks WITH replication = {'class': 'SimpleStrategy' , 'replication_factor' : '3'};

cqlsh:CREATE TABLE dataks.datatable (
    key text PRIMARY KEY,
    bindata boolean,
    intdata int,
    txtdata text
)
```
Now we have the schema created in the Cassandra node, let's add some data to the table

```
cqlsh:dataks> INSERT INTO datatable (key , intdata , txtdata , bindata ) VALUES ( 'key1', 1, 'txt1', false);
cqlsh:dataks> INSERT INTO datatable (key , intdata , txtdata , bindata ) VALUES ( 'key2', 2, 'txt2', false);
cqlsh:dataks> INSERT INTO datatable (key , intdata , txtdata , bindata ) VALUES ( 'key3', 3, 'txt3', false);
cqlsh:dataks> INSERT INTO datatable (key , intdata , txtdata , bindata ) VALUES ( 'key4', 4, 'txt4', true);
cqlsh:dataks> INSERT INTO datatable (key , intdata , txtdata , bindata ) VALUES ( 'key5', 5, 'txt5', true);
cqlsh:dataks> INSERT INTO datatable (key , intdata , txtdata , bindata ) VALUES ( 'key6', 6, 'txt6', true);
cqlsh:dataks> INSERT INTO datatable (key , intdata , txtdata , bindata ) VALUES ( 'key7', 7, 'txt7', false);
cqlsh:dataks> INSERT INTO datatable (key , intdata , txtdata , bindata ) VALUES ( 'key8', 8, 'txt8', true);
```



