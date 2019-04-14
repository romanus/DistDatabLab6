## Task 6 - Cassandra Replication

by Teodor Romanus

#### 1. Configure 3-node cluster

```
docker pull cassandra
docker run --name cas1 -p 9042:9042 -e CASSANDRA_CLUSTER_NAME=MyCluster -d cassandra
docker run --name cas2 -e CASSANDRA_SEEDS="$(docker inspect --format='{{ .NetworkSettings.IPAddress }}' cas1)" -e CASSANDRA_CLUSTER_NAME=MyCluster -d cassandra
docker run --name cas3 -e CASSANDRA_SEEDS="$(docker inspect --format='{{ .NetworkSettings.IPAddress }}' cas1)" -e CASSANDRA_CLUSTER_NAME=MyCluster -d cassandra
```

```
macbook893:DistDatabLab6 trom$ docker pull cassandra
Using default tag: latest
latest: Pulling from library/cassandra
Digest: sha256:9f1d47fd23261c49f226546fe0134e6d4ad0570b7ea3a169c521005cb8369a32
Status: Image is up to date for cassandra:latest
macbook893:DistDatabLab6 trom$ docker run --name cas1 -p 9042:9042 -e CASSANDRA_CLUSTER_NAME=MyCluster -e CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch -e CASSANDRA_DC=datacenter1 -d cassandra
56899e0e7c1c8974c929fb556e2aff764be89922617eb38a4effde52a241bd82
macbook893:DistDatabLab6 trom$ docker run --name cas2 -e CASSANDRA_SEEDS="$(docker inspect --format='{{ .NetworkSettings.IPAddress }}' cas1)" -e CASSANDRA_CLUSTER_NAME=MyCluster -e CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch -e CASSANDRA_DC=datacenter2 -d cassandra
9c57eacd318678450db243fb61e6518202734426470fdfbe27e99b9cbb75fa76
macbook893:DistDatabLab6 trom$ docker run --name cas3 -e CASSANDRA_SEEDS="$(docker inspect --format='{{ .NetworkSettings.IPAddress }}' cas1)" -e CASSANDRA_CLUSTER_NAME=MyCluster -e CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch -e CASSANDRA_DC=datacenter3 -d cassandra
1f1624d1f20fd08973376530246782e2cbe78811d9a285665aa120c7be5ec843
```

#### 2. Check configuration

```
docker exec -ti cas1 nodetool status
```

```
macbook893:DistDatabLab6 trom$ docker exec -ti cas1 nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID                               Rack
UN  172.17.0.3  93.98 KiB  256          69.0%             b683836c-e348-430c-90ae-45460561427e  rack1
UN  172.17.0.2  108.6 KiB  256          67.8%             fe564ca3-3f4d-42e8-a372-7defadb1d47d  rack1
UN  172.17.0.4  30.06 KiB  256          63.1%             c5594641-3e8c-4e4d-877a-b9cd45512fa6  rack1
```

#### 3. Create keyspaces

```
docker exec -ti cas1 cqlsh
CREATE KEYSPACE keyspace1 WITH replication = { 'class': 'NetworkTopologyStrategy', 'datacenter1':1 };
CREATE KEYSPACE keyspace2 WITH replication = { 'class': 'NetworkTopologyStrategy', 'datacenter1':2 };
CREATE KEYSPACE keyspace3 WITH replication = { 'class': 'NetworkTopologyStrategy', 'datacenter1':3 };
```

```
macbook893:DistDatabLab6 trom$ docker exec -ti cas1 cqlsh
Connected to MyCluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.4 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh> CREATE KEYSPACE keyspace1 WITH replication = { 'class': 'NetworkTopologyStrategy', 'datacenter1':1 };
cqlsh> CREATE KEYSPACE keyspace2 WITH replication = { 'class': 'NetworkTopologyStrategy', 'datacenter1':2 };
cqlsh> CREATE KEYSPACE keyspace2 WITH replication = { 'class': 'NetworkTopologyStrategy', 'datacenter1':2 };
```

#### 4. Create tables

```
CREATE TABLE keyspace1.table1 (id INT PRIMARY KEY, name TEXT);
CREATE TABLE keyspace2.table2 (id INT PRIMARY KEY, name TEXT);
CREATE TABLE keyspace3.table3 (id INT PRIMARY KEY, name TEXT);
```

```
cqlsh> CREATE TABLE keyspace1.table1 (id INT PRIMARY KEY, name TEXT);
cqlsh> CREATE TABLE keyspace2.table2 (id INT PRIMARY KEY, name TEXT);
cqlsh> CREATE TABLE keyspace3.table3 (id INT PRIMARY KEY, name TEXT);
```

#### 5. Write and Read to/from different nodes

```
INSERT INTO keyspace1.table1 (id, name) VALUES (1, '1');
INSERT INTO keyspace1.table1 (id, name) VALUES (2, '2');
INSERT INTO keyspace2.table2 (id, name) VALUES (1, '1');
INSERT INTO keyspace2.table2 (id, name) VALUES (2, '2');
INSERT INTO keyspace3.table3 (id, name) VALUES (1, '1');
INSERT INTO keyspace3.table3 (id, name) VALUES (2, '2');
exit
docker exec -ti cas2 cqlsh
SELECT * FROM keyspace1.table1;
```

```
cqlsh> INSERT INTO keyspace1.table1 (id, name) VALUES (1, '1');
cqlsh> INSERT INTO keyspace1.table1 (id, name) VALUES (2, '2');
cqlsh> INSERT INTO keyspace2.table2 (id, name) VALUES (1, '1');
cqlsh> INSERT INTO keyspace2.table2 (id, name) VALUES (2, '2');
cqlsh> INSERT INTO keyspace3.table3 (id, name) VALUES (1, '1');
cqlsh> INSERT INTO keyspace3.table3 (id, name) VALUES (2, '2');
cqlsh> exit
macbook893:DistDatabLab6 trom$ docker exec -ti cas2 cqlsh
Connected to MyCluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.4 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh> SELECT * FROM keyspace1.table1;

 id | name
----+------
  1 |    1
  2 |    2

(2 rows)
```

#### 6. Look into data distribution at the nodes

```
docker exec -ti cas1 nodetool status keyspace1
docker exec -ti cas1 nodetool status keyspace2
docker exec -ti cas1 nodetool status keyspace3
```

```
macbook893:DistDatabLab6 trom$ docker exec -ti cas1 nodetool status keyspace1
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID                               Rack
UN  172.17.0.3  92.31 KiB  256          33.9%             b683836c-e348-430c-90ae-45460561427e  rack1
UN  172.17.0.2  108.91 KiB  256          32.7%             fe564ca3-3f4d-42e8-a372-7defadb1d47d  rack1
UN  172.17.0.4  70.45 KiB  256          33.4%             c5594641-3e8c-4e4d-877a-b9cd45512fa6  rack1

macbook893:DistDatabLab6 trom$ docker exec -ti cas1 nodetool status keyspace2
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID                               Rack
UN  172.17.0.3  92.31 KiB  256          69.0%             b683836c-e348-430c-90ae-45460561427e  rack1
UN  172.17.0.2  108.91 KiB  256          67.8%             fe564ca3-3f4d-42e8-a372-7defadb1d47d  rack1
UN  172.17.0.4  70.45 KiB  256          63.1%             c5594641-3e8c-4e4d-877a-b9cd45512fa6  rack1

macbook893:DistDatabLab6 trom$ docker exec -ti cas1 nodetool status keyspace3
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID                               Rack
UN  172.17.0.3  92.31 KiB  256          100.0%            b683836c-e348-430c-90ae-45460561427e  rack1
UN  172.17.0.2  108.91 KiB  256          100.0%            fe564ca3-3f4d-42e8-a372-7defadb1d47d  rack1
UN  172.17.0.4  70.45 KiB  256          100.0%            c5594641-3e8c-4e4d-877a-b9cd45512fa6  rack1
```

#### 7. Look where data from different tables and keyspaces is saved

```
docker exec -ti cas1 nodetool getendpoints -- keyspace1 table1 1
docker exec -ti cas1 nodetool getendpoints -- keyspace2 table2 1
docker exec -ti cas1 nodetool getendpoints -- keyspace3 table3 1
```

```
macbook893:DistDatabLab6 trom$ docker exec -ti cas1 nodetool getendpoints -- keyspace1 table1 1
172.17.0.4
macbook893:DistDatabLab6 trom$ docker exec -ti cas1 nodetool getendpoints -- keyspace2 table2 1
172.17.0.4
172.17.0.2
macbook893:DistDatabLab6 trom$ docker exec -ti cas1 nodetool getendpoints -- keyspace3 table3 1
172.17.0.4
172.17.0.2
172.17.0.3
```

#### 8. Change consistencies

```
???
```

#### 9. Disconnect nodes

```
???
```

#### 10. Write different values to different nodes

```
docker exec -ti cas1 cqlsh
CONSISTENCY ONE
INSERT INTO keyspace3.table3 (id, name) VALUES (3, '3');
exit
docker exec -ti cas2 cqlsh
CONSISTENCY ONE
INSERT INTO keyspace3.table13(id, name) VALUES (3, '4');
exit
docker exec -ti cas3 cqlsh
CONSISTENCY ONE
INSERT INTO keyspace3.table13(id, name) VALUES (3, '5');
exit
```

```
macbook893:DistDatabLab6 trom$ docker exec -ti cas1 cqlsh
Connected to MyCluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.4 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh> CONSISTENCY ONE
Consistency level set to ONE.
cqlsh> INSERT INTO keyspace3.table3 (id, name) VALUES (3, '3');
cqlsh> exit
macbook893:DistDatabLab6 trom$ docker exec -ti cas2 cqlsh
Connected to MyCluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.4 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh> CONSISTENCY ONE
Consistency level set to ONE.
cqlsh> INSERT INTO keyspace3.table3 (id, name) VALUES (3, '4');
cqlsh> exit
macbook893:DistDatabLab6 trom$ docker exec -ti cas3 cqlsh
Connected to MyCluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.4 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh> CONSISTENCY ONE
Consistency level set to ONE.
cqlsh> INSERT INTO keyspace3.table3 (id, name) VALUES (3, '5');
cqlsh> exit
```

#### 11. Join info from nodes

```
???
```