# Migration to Scylla
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

# Migrating from Cassandra to Scylla using COPY command

From CQLSH shell on the Cassandra node use the copy command:
```
COPY datatable TO '/var/lib/cassandra/myfile.csv'
```
In the node’s data directory (from which you executed the COPY command) you will find the csv file.
```
cat myfile.csv
key7,False,7,txt7
key8,True,8,txt8
key2,False,2,txt2
key3,False,3,txt3
key1,False,1,txt1
key4,True,4,txt4
key5,True,5,txt5
key6,True,6,txt6
```

As you can see we do not have any header to the csv file. To get the column names populated we will add the HEADER option.

`COPY datatable TO '/var/lib/cassandra/myfilewithheader.csv' WITH HEADER=TRUE;`
The file will look as the following:
```
cat myfilewithheader.csv
key,bindata,intdata,txtdata
key7,False,7,txt7
key4,True,4,txt4
key5,True,5,txt5
key6,True,6,txt6
key8,True,8,txt8
key3,False,3,txt3
key2,False,2,txt2
key1,False,1,txt1
```
To load the data into the Scylla cluster we will copy the file to the Scylla data directory of one of our nodes, and execute the `COPY from` command.
But, first we need to create a keyspace and datatable, please note that keyspace names can be modified, column names can be modified too. However, data types and order must be kept.
```
cqlsh> CREATE KEYSPACE scylladataks WITH replication = {'class': 'SimpleStrategy' , 'replication_factor' : '3'};
cqlsh> CREATE TABLE scylladataks.scylladatatable (
   ...     key text PRIMARY KEY,
   ...     bindata boolean,
   ...     intdata int,
   ...     txtdata text
   ... );

COPY scylladataks.scylladatatable FROM '/var/lib/scylla/data/myfilewithheader.csv' WITH HEADER=TRUE;
Reading options from the command line: {'header': 'TRUE'}
Using 3 child processes

cqlsh> SELECT * FROM scylladataks.scylladatatable ;

 key  | bindata | intdata | txtdata
------+---------+---------+---------
 key5 |    True |       5 |    txt5
 key6 |    True |       6 |    txt6
 key1 |   False |       1 |    txt1
 key4 |    True |       4 |    txt4
 key8 |    True |       8 |    txt8
 key3 |   False |       3 |    txt3
 key7 |   False |       7 |    txt7
 key2 |   False |       2 |    txt2

(8 rows)
```

Please note, write times, TTLs and any metadata item are not part of the data transferred using the CQL COPY command.


# Using SStableloader 

Prior to starting the migration, In the Cassandra cluster, flush and compact the data:
```
nodetool flush
nodetool compact
```
Next, create a snapshot of the data with:
```
nodetool snapshot dataks

Requested creating snapshot(s) for [dataks] with snapshot name [1572731721859] and options {skipFlush=false}
Snapshot directory: 1572731721859
 
The files snapshot will be created in a folder with similar name (UUID and snapshot timestamp will be different):
/var/lib/cassandra/data/dataks/datatable-75fbc4f0fd8611e9a051d335f841c590/snapshots/1572731721859/
```
And the content of the directory will look, like the following:
```
-rw-r--r-- 1 root root  867 Nov  2 23:26 schema.cql
-rw-r--r-- 2 root root   92 Nov  2 23:26 mc-1-big-TOC.txt
-rw-r--r-- 2 root root   56 Nov  2 23:26 mc-1-big-Summary.db
-rw-r--r-- 2 root root 4735 Nov  2 23:26 mc-1-big-Statistics.db
-rw-r--r-- 2 root root   68 Nov  2 23:26 mc-1-big-Index.db
-rw-r--r-- 2 root root   24 Nov  2 23:26 mc-1-big-Filter.db
-rw-r--r-- 2 root root   10 Nov  2 23:26 mc-1-big-Digest.crc32
-rw-r--r-- 2 root root  184 Nov  2 23:26 mc-1-big-Data.db
-rw-r--r-- 2 root root   43 Nov  2 23:26 mc-1-big-CompressionInfo.db
-rw-r--r-- 1 root root   31 Nov  2 23:26 manifest.json
```

We will now transfer the data files to an intermediate node we will add to the Scylla network with the following docker command:

```
mkdir /var/lib/scylla/data/dataks/datatable/
sh-4.2# cp -p  /var/lib/cassandra_data/data/dataks/datatable-75fbc4f0fd8611e9a051d335f841c590/snapshots/1572731721859/* /var/lib/scylla/data/dataks/datatable/.
sh-4.2# ls -lrt /var/lib/scylla/data/dataks/datatable/
total 44
-rw-r--r-- 1 scylla root  867 Nov  2 23:26 schema.cql
-rw-r--r-- 1 scylla root   92 Nov  2 23:26 mc-1-big-TOC.txt
-rw-r--r-- 1 scylla root   56 Nov  2 23:26 mc-1-big-Summary.db
-rw-r--r-- 1 scylla root 4735 Nov  2 23:26 mc-1-big-Statistics.db
-rw-r--r-- 1 scylla root   68 Nov  2 23:26 mc-1-big-Index.db
-rw-r--r-- 1 scylla root   24 Nov  2 23:26 mc-1-big-Filter.db
-rw-r--r-- 1 scylla root   10 Nov  2 23:26 mc-1-big-Digest.crc32
-rw-r--r-- 1 scylla root  184 Nov  2 23:26 mc-1-big-Data.db
-rw-r--r-- 1 scylla root   43 Nov  2 23:26 mc-1-big-CompressionInfo.db
-rw-r--r-- 1 scylla root   31 Nov  2 23:26 manifest.json
```
We will take the schema from the Cassandra nodes with: 
On a cassandra node do: 
```
cqlsh [Cassandra node IP] "-e DESC SCHEMA" > /var/lib/cassandra/data/origschema.cql
```
The above command will export the schema, we do need to make changes in the original schema.
In general, we would like to modify the `min_threshold` for compaction to 2

Now we will copy the modified schema into the Scylla node by executing the following command on the Scylla contrainer
```
cqlsh [Scylla node IP] --file /var/lib/scylla/data/adjustedschema.cql
```
Let's make sure the schema is uploaded properly

```
cqlsh [Scylla node IP]
Connected to  at [Scylla node IP]:9042.
[cqlsh 5.0.1 | Cassandra 3.0.8 | CQL spec 3.3.1 | Native protocol v4]
Use HELP for help.
cqlsh> DESC SCHEMA ;

CREATE KEYSPACE scylladataks WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '3'}  AND durable_writes = true;

CREATE TABLE scylladataks.scylladatatable (
    key text PRIMARY KEY,
    bindata boolean,
    intdata int,
    txtdata text
) WITH bloom_filter_fp_chance = 0.01
    AND caching = {'keys': 'ALL', 'rows_per_partition': 'ALL'}
    AND comment = ''
    AND compaction = {'class': 'SizeTieredCompactionStrategy'}
    AND compression = {'sstable_compression': 'org.apache.cassandra.io.compress.LZ4Compressor'}
    AND crc_check_chance = 1.0
    AND dclocal_read_repair_chance = 0.1
    AND default_time_to_live = 0
    AND gc_grace_seconds = 864000
    AND max_index_interval = 2048
    AND memtable_flush_period_in_ms = 0
    AND min_index_interval = 128
    AND read_repair_chance = 0.0
    AND speculative_retry = '99.0PERCENTILE';

CREATE KEYSPACE dataks WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '3'}  AND durable_writes = true;

CREATE TABLE dataks.datatable (
    key text PRIMARY KEY,
    bindata boolean,
    intdata int,
    txtdata text
) WITH bloom_filter_fp_chance = 0.01
    AND caching = {'keys': 'ALL', 'rows_per_partition': 'ALL'}
    AND comment = ''
    AND compaction = {'class': 'SizeTieredCompactionStrategy'}
    AND compression = {'sstable_compression': 'org.apache.cassandra.io.compress.LZ4Compressor'}
    AND crc_check_chance = 1.0
    AND dclocal_read_repair_chance = 0.1
    AND default_time_to_live = 0
    AND gc_grace_seconds = 864000
    AND max_index_interval = 2048
    AND memtable_flush_period_in_ms = 0
    AND min_index_interval = 128
    AND read_repair_chance = 0.0
    AND speculative_retry = '99.0PERCENTILE';
```

As you can see, we have created a new keyspace and table with the original names we used in the Cassandra node. The data we uploaded with the CQL copy command is present as well,  in a different keyspace and table.
Now let’s load the data using our intermediate container. 
`docker exec -it intermediate sh` and execute the following command
```
sh-4.2# sstableloader -d [Scylla node IP]  /var/lib/scylla/data/dataks/datatable
WARN  01:08:48,328 dclocal_read_repair_chance and read_repair_chance table options have been deprecated and will be removed in version 4.0
100% done.        8 statements sent (in        7 batches,        0 failed).
       8 statements generated.
      16 cql rows processed in        8 partitions.
       0 cql rows and        0 partitions deleted.
       0 local and        0 remote counter shards where skipped.
```
Data is uploaded into the Scylla node.
Back to our Scylla node, let’s verify the data is there:
```
cqlsh> select * FROM dataks.datatable;

 key  | bindata | intdata | txtdata
------+---------+---------+---------
 key5 |    True |       5 |    txt5
 key6 |    True |       6 |    txt6
 key1 |   False |       1 |    txt1
 key4 |    True |       4 |    txt4
 key8 |    True |       8 |    txt8
 key3 |   False |       3 |    txt3
 key7 |   False |       7 |    txt7
 key2 |   False |       2 |    txt2

(8 rows)
cqlsh> SELECT writetime(bindata) FROM dataks.datatable WHERE key='key6';

 writetime(bindata)
--------------------
   1572709068911098

(1 rows)
```

With SSTableloader you are able to copy the columns metadata, which will help you save the TTL and writetime of the different columns. 


For Spark Migrator example. The following link has an excellent example:
https://github.com/scylladb/scylla-migrator#running-locally


