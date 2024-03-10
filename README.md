# Cassandra Cluster Playground

This project will demo several key concepts in Cassandra NoSQL database such as replication and tunble consistency level.

Good news is that everything can be done locally on your Mac.

## Cassandra Architecture
Cassandra is a distributed NoSQL database system designed for handling large amounts of data across multiple nodes while providing high availability and fault tolerance. Here's an overview of its architecture:
* *Decentralized Architecture*: Cassandra follows a decentralized architecture where there is no single point of failure. Each node in the Cassandra cluster is identical and self-sufficient, with no leader-follower relationship.
* *Peer-to-Peer Communication*: Nodes communicate with each other using a peer-to-peer protocol. Each node can accept read and write requests, making the system highly scalable.
* *Replication*: Cassandra replicates data across multiple nodes to ensure high availability and fault tolerance. Replication is configurable at the keyspace level, allowing users to define the replication factor and replication strategy.
* *Data Distribution*: Cassandra distributes data across nodes using a partitioning scheme based on consistent hashing. Each piece of data is assigned a partition key, and Cassandra uses the partition key to determine which node should store the data. This ensures even distribution of data across the cluster.
* *CAP Theorem*: Cassandra is designed to provide high availability and partition tolerance (AP) while sacrificing strong consistency (CP) in certain scenarios. It offers tunable consistency levels, allowing users to choose the level of consistency required for their applications.
* Gossip Protocol*: Cassandra uses a gossip protocol for node discovery, failure detection, and membership management. Nodes periodically exchange information about the cluster's state, allowing them to detect and react to changes dynamically.
* Data Model*: Cassandra uses a column-family data model. Data is organized into keyspaces, which contain tables. Each table consists of rows and columns, where rows are identified by a primary key. Tables can have multiple columns and support sparse data, allowing each row to have a different set of columns.
* Query Language*: Cassandra Query Language (CQL) is the primary interface for interacting with Cassandra. CQL is similar to SQL but optimized for the distributed nature of Cassandra. It supports a subset of SQL operations for creating, reading, updating, and deleting data.

In summary, Cassandra's architecture is designed to provide **scalability**, **high availability**, and **fault tolerance** by distributing data across multiple nodes in a decentralized manner. Its decentralized and peer-to-peer nature, coupled with configurable replication and partitioning, makes it suitable for handling large-scale distributed data storage and processing requirements.

![Architecture1](https://github.com/pdesai5839/cassandra_cluster/assets/143283961/36cb55cb-7adf-4cfd-9b0f-38c35e6c8b1a)

(Image Credit: geeksforgeeks.org)

## Install Cassandra

`brew install cassandra`

By default, Cassandra uses port 7000 for cluster communication (port 7001 if SSL is enabled), port 9042 for native protocol clients, and port 7199 for JMX.

Let's use Docker to fire up a 3 node cluster using Cassandra 3.11. Note that this could take a while. Save the snippet below as `start.sh` and run it from a terminal.

```shell
# Run the first node
docker run --name cassandra-1 -p 9042:9042 -d cassandra:3.11
INSTANCE1=$(docker inspect --format="{{ .NetworkSettings.IPAddress }}" cassandra-1)
echo "Instance 1: ${INSTANCE1}"

# Run the second node
docker run --name cassandra-2 -p 9043:9042 -d -e CASSANDRA_SEEDS=$INSTANCE1 cassandra:3.11
INSTANCE2=$(docker inspect --format="{{ .NetworkSettings.IPAddress }}" cassandra-2)
echo "Instance 2: ${INSTANCE2}"

echo "Waiting 60s for the second node to join the cluster"
sleep 60

# Run the third node
docker run --name cassandra-3 -p 9044:9042 -d -e CASSANDRA_SEEDS=$INSTANCE1,$INSTANCE2 cassandra:3.11
INSTANCE3=$(docker inspect --format="{{ .NetworkSettings.IPAddress }}" cassandra-3)
```

## Verify

If all goes well, our 3 node cluster should be up and running.
<img width="707" alt="Screenshot 2024-03-02 at 2 46 24 PM" src="https://github.com/pdesai5839/cassandra_cluster/assets/143283961/6b297d33-07f1-4217-be96-cb043b5b457b">

Let's verify by running the following:

`docker exec cassandra-3 nodetool status`

The output should be similar to this:
```
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID                               Rack
UN  172.17.0.3  70.87 KiB  256          63.7%             71312be9-68ee-4e31-ac14-552d9b158914  rack1
UN  172.17.0.2  75.9 KiB   256          70.5%             da66c9b9-b30d-48b9-b87b-63382a036dcd  rack1
UN  172.17.0.4  70.89 KiB  256          65.8%             166fd946-03c3-4ba2-9030-4d481799537f  rack1
```

Note the "Owns" column. This is the percentage of the data owned by the node per datacenter times the replication factor.

"Tokens" can be thought of as "token ranges". Each of the hosts has 256 different token ranges. Cassandra does this to distribute data evenly as the cluster grows. A lower number means less-even distribution.

## CQLSH

Cqlsh (Cassandra Query Language Shell) is a command-line interface (CLI) for interacting with Cassandra databases using the Cassandra Query Language (CQL). It allows users to execute CQL statements and commands to perform various database operations such as creating keyspaces, tables, and indexes, inserting, updating, and deleting data, as well as querying data from Cassandra tables.

Now that we know what cqlsh is, let's try it out!

Connect to the first node by starting cqlsh shell in Docker:

`docker exec -it cassandra-1 cqlsh`

When connected, you'll see this:

```shell
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.16 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh>
```

## Keyspaces

Time to execute our first query. Run the `DESCRIBE` command:
```sql
DESCRIBE keyspaces;

system_traces  system_schema  system_auth  system  system_distributed
```

The `DESCRIBE keyspaces;` command is used to display information about the keyspaces present in the Cassandra database cluster.

So what are keyspaces anyway? Let's define it in simple terms:
Keyspaces are the highest-level containers for organizing data. Think of the more familiar Schema/Database in traditional SQL RDMS and you'll get a feel for what Keyspaces are. Keyspaces are top-level containers for organizing data. However, this is where the similarity ends.

So knowing what keyspaces are, we must create the keyspaces first in order to create tables. We'll create a simple table with some geo data to play around with Cassandra.  

```sql
CREATE KEYSPACE geo_data 
  WITH REPLICATION = { 
   'class' : 'NetworkTopologyStrategy',
   'datacenter1' : 3 
};
```

What did we just do? We just created a keyspace named `geo_data` with these replication settings:
* The replication strategy specified is "NetworkTopologyStrategy". This strategy is used to replicate data across multiple data centers in a Cassandra cluster.
* Within the "NetworkTopologyStrategy", the replication factor for the "datacenter1" is set to 3. This means that the data in this keyspace will be replicated across 3 nodes within the "datacenter1" data center.

# Importance of Replication Factor

In Cassandra, setting a replication factor of 3 is a common practice for several reasons:
* *Fault Tolerance*: Cassandra is designed to be highly fault-tolerant. By replicating data across multiple nodes (in this case, three nodes), it ensures that there are multiple copies of each piece of data available. If one node goes down or becomes unavailable, the data can still be accessed from the other replicas, ensuring high availability.
* *Consistency*: Cassandra uses a distributed consensus protocol to ensure data consistency. With a replication factor of 3, Cassandra can achieve strong consistency guarantees while still providing high availability. If one node becomes unavailable, Cassandra can use the remaining replicas to maintain data consistency.
* *Read and Write Performance*: Replicating data across multiple nodes can improve read and write performance. With multiple replicas available, Cassandra can distribute read and write requests across these replicas, reducing the load on individual nodes and improving overall system performance.
* *Data Distribution*: Cassandra uses a decentralized architecture where data is distributed across multiple nodes in the cluster. A replication factor of 3 helps ensure that data is evenly distributed across the cluster, preventing hotspots and ensuring efficient data access.

* ![mwpqwzenp3bb164vwz4m](https://github.com/pdesai5839/cassandra_cluster/assets/143283961/620314ed-6d24-423f-a2da-90fa2d3e9941)

(Image Credit: dev.to)

## Coordinator Node / Consistent Hashing

In Cassandra, you can make reads and writes to any node in the cluster. Cassandra will route this request to the correct node, meaning you don’t need to worry about what node you are connected to versus which node actually has the data. The coordinator node is responsible for determining which nodes in the cluster are responsible for the requested data (based on the partition key) and routing the request to those nodes. How does the coordinator figure out which node contains the data? By using the consistent hashing algorithm.

Consistent hashing provides a way to efficiently distribute and balance data across multiple nodes in the system while minimizing the impact of node additions or removals.

Here's how consistent hashing works:

1. **Hash Ring**:
   * Imagine a hash ring, which is a circular ring consisting of a large number of hash values arranged in a circular manner.
   
   ![1_ztjT3sM_O7mLDGL9l6g21w copy](https://github.com/pdesai5839/cassandra_cluster/assets/143283961/dcedd829-3f25-46a4-a59a-d7740e5545a1)

   (Image Credit: medium.com)

2. **Node Mapping**:
   * Each node is assigned a unique identifier (such as an IP address or a node ID). This identifier is hashed to generate a hash value, which is then mapped onto the hash ring.
   
   ![1_N0Wu97jOKBjDD4wRz0GnWg copy](https://github.com/pdesai5839/cassandra_cluster/assets/143283961/d7024a6d-8501-4a08-b55e-afcd10ba4aa5)

   (Image Credit: medium.com)

3. **Data Partitioning**:
   * Data keys are also hashed to generate hash values. These hash values are then mapped onto the hash ring in the same manner as the node identifiers.
   
   ![1666715227884](https://github.com/pdesai5839/cassandra_cluster/assets/143283961/6b8b54cd-5cf0-41d2-93e0-b551ee7041eb)

   (Image Credit: linkedin.com)

4. **Data Placement**:
   * To find the node responsible for storing a particular piece of data, the system locates the node whose identifier is the next highest one on the hash ring from the hash value of the data key. This is done by traversing the hash ring in a clockwise direction until the next highest node is found.
   
5. **Consistent Hashing Property**:
   * The key property of consistent hashing is that when a node is added or removed from the system, only a fraction of the data needs to be remapped. Specifically, when a node is added or removed, only the data that would have been assigned to or previously assigned to that node and its immediate successor on the hash ring need to be remapped. This minimizes the amount of data movement required, even when the number of nodes in the system changes.

## Data Partitioning

Data partitioning is a key aspect of Cassandra's architecture. It allowsdistribution of data across multiple nodes in a cluster for scalability and fault tolerance. Here's how data partitioning works in Cassandra:

1. **Partition Key**:
   * Each row in a Cassandra table is uniquely identified by a primary key, which consists of one or more columns. The first column in the primary key is known as the partition key.
   * The partition key is used to determine which node in the cluster will store the data for that row.

2. **Token-Based Partitioning**:
   * Cassandra uses a token-based partitioning scheme to distribute data across nodes in the cluster. It generates a token (a 128-bit integer) for each partition key value using a hash function (usually Murmur3).
   * The token value determines the placement of data on the hash ring, which is a logical representation of the nodes in the cluster.

3. **Partitioner**:
   * Cassandra uses a partitioner to map partition key values to tokens and determine which nodes are responsible for storing the data.
   * The partitioner ensures that each token is assigned to a specific node in the cluster based on the token's position on the hash ring.

4. **Data Distribution**:
   * When data is written to Cassandra, the partitioner hashes the partition key value to generate a token. Cassandra then uses the token to determine which node in the cluster will store the data.
   * Each node is responsible for storing data for a range of tokens, and each token corresponds to a range of partition key values.
   * Cassandra ensures that data is evenly distributed across nodes in the cluster to prevent hotspots and ensure efficient data access.

5. **Replication**:
   * To ensure fault tolerance and high availability, Cassandra replicates data across multiple nodes in the cluster. Each piece of data is replicated to a configurable number of replica nodes (determined by the replication factor) using a replication strategy specified at the keyspace level.
   * By using partitioning and replication, Cassandra can distribute and replicate data across multiple nodes in the cluster, providing scalability, fault tolerance, and high availability for large-scale distributed data storage and processing applications.

## Partition Key/Primary Key

In Cassandra, tables are defined with columns and corresponding data types. Each table must have a primary key, which uniquely identifies rows based on one or multiple columns.

However, unlike traditional databases such as MySQL, defining a primary key in Cassandra involves two main components:

1. A mandatory partition key: This key is required and serves as the primary identifier for data partitioning.
2. An optional set of clustering columns: These columns, if specified, help define the sorting order within each partition.

Consider this table containing region data:

| country | region | region_name | region_code | timezone            |
|---------|--------|-------------|-------------|---------------------|
| fra     | bre    | bretagne    | 34974       | Europe/Paris        |
| fra     | nor    | normandie   | 34980       | Europe/Paris        |
| usa     | ca     | california  | 5           | America/Los_Angeles |
| usa     | co     | colorado    | 6           | America/Denver      |
| usa     | va     | virginia    | 47          | America/New_York    |

Here, we will choose `country` as the partition key and `region_name` as the clustering key. A combination of both `country` and `region_name` will make up the primary key.

Now, let us proceed to create the table using the CREATE TABLE construct:
```sql
CREATE TABLE geo_data.regions_by_country (
    country text,
    region text,
    region_name text,
    region_code int,
    timezone text,
    PRIMARY KEY ((country), region_name)
);
```

Note that the primary key consists of the partition key and the clustering key. The first group of the primary key specifies the partition key. All other parts of the primary key is one or more clustering keys. The Primary Key defines what columns are used to identify rows. Add all columns that are required to identify a row uniquely to the primary key. In our sample data, using just the `country` column would not uniquely identify each row, which is why we added `region_name` column to the primary key.

We'll now populate the table with the sample data using cqlsh:

```sql
BEGIN BATCH 

INSERT INTO geo_data.regions_by_country (country, region, region_name, region_code, timezone)
  VALUES('fra','bre','bretagne',34974,'Europe/Paris');

INSERT INTO geo_data.regions_by_country (country, region, region_name, region_code, timezone)
  VALUES('fra','nor','normandie',34980,'Europe/Paris');

INSERT INTO geo_data.regions_by_country (country, region, region_name, region_code, timezone)
  VALUES('usa','ca','california',5,'America/Los_Angeles');

INSERT INTO geo_data.regions_by_country (country, region, region_name, region_code, timezone)
  VALUES('usa','co','colorado',6,'America/Denver');

INSERT INTO geo_data.regions_by_country (country, region, region_name, region_code, timezone)
  VALUES('usa','va','virginia',47,'America/New_York');

APPLY BATCH;
```

Let's verify that our data is populated in the table:

```sql
SELECT * FROM geo_data.regions_by_country;
```
```shell
 country | region_name | region | region_code | timezone
---------+-------------+--------+-------------+---------------------
     fra |    bretagne |    bre |       34974 |        Europe/Paris
     fra |   normandie |    nor |       34980 |        Europe/Paris
     usa |  california |     ca |           5 | America/Los_Angeles
     usa |    colorado |     co |           6 |      America/Denver
     usa |    virginia |     va |          47 |    America/New_York
```

## Efficient Partitioning
The `regions_by_country` table uses `country` as the Partition Key. What does this mean? It means that all rows matching a country value will be placed in the same partition.

Here's an illustration of how the data would be partitioned:
![cassandra partition (2)](https://github.com/pdesai5839/cassandra_cluster/assets/143283961/7a722162-1c76-49c8-aa5a-c299e10045ad)


Let's note a few things:
1. Rows in each partition are ordered by the clustering key.
2. Combination of partition key & clustering key uniquely identifies each row.
3. Partition key is used to create and populate the partitions.
4. Data is read from and written to different nodes based on the partiton key.

It may be apparent by now that it is super important to understand the distribution of data.
We must carefully consider how the data is read and writitten among the partitions.

The partition key helps distribute data evenly between nodes, it is also needed when reading the data.

Our schema for regions is designed to be queried by the partition key which is `country`.

```sql
SELECT * FROM geo_data.regions_by_country WHERE country = 'usa';
```
```shell
 country | region_name | region | region_code | timezone
---------+-------------+--------+-------------+---------------------
     usa |  california |     ca |           5 | America/Los_Angeles
     usa |    colorado |     co |           6 |      America/Denver
     usa |    virginia |     va |          47 |    America/New_York
```

This query would have been sent to a single node by default. This is known as consistency level of one.

## So, what exactly is consistency level of one?

A consistency level of one indicates that a read or write operation must be acknowledged by only one replica node. When a client performs a read or write operation with a consistency level of one, Cassandra sends the request to one replica node responsible for the data partition and waits for an acknowledgment from that node. Once the acknowledgment is received, the operation is considered successful, even if the data has not been replicated to other nodes in the cluster.

While a consistency level of one offers low latency and high availability for read and write operations, it sacrifices consistency guarantees. In scenarios where consistency is not critical or where low latency is prioritized over consistency, using a consistency level of one may be appropriate. However, it's essential to consider the trade-offs and potential implications on data consistency when choosing the consistency level for operations in Cassandra.

## Inefficient Partitioning

Let's create another table and populate it to illustrate an inefficient partition key that will require Cassandra to gather data from multiple partitions.

```sql
CREATE TABLE geo_data.regions_by_code (
    country text,
    region text,
    region_name text,
    region_code int,
    timezone text,
    PRIMARY KEY (region_code)
);
```
```sql
BEGIN BATCH 

INSERT INTO geo_data.regions_by_code (country, region, region_name, region_code, timezone)
  VALUES('fra','bre','bretagne',34974,'Europe/Paris');

INSERT INTO geo_data.regions_by_code (country, region, region_name, region_code, timezone)
  VALUES('fra','nor','normandie',34980,'Europe/Paris');

INSERT INTO geo_data.regions_by_code (country, region, region_name, region_code, timezone)
  VALUES('usa','ca','california',5,'America/Los_Angeles');

INSERT INTO geo_data.regions_by_code (country, region, region_name, region_code, timezone)
  VALUES('usa','co','colorado',6,'America/Denver');

INSERT INTO geo_data.regions_by_code (country, region, region_name, region_code, timezone)
  VALUES('usa','va','virginia',47,'America/New_York');

APPLY BATCH;
```

Assume Cassandra assigns the above rows as shown here:
![cassandra partition 2](https://github.com/pdesai5839/cassandra_cluster/assets/143283961/3dc3b2cc-3515-4d1d-a81a-4633edb77c9c)

This new table uses `region_code` as the partition key. If the use case requires us to get region data by its code, then the following query will work efficiently because Cassandra only needs to read from 1 partition for the data:

```sql
SELECT * FROM geo_data.regions_by_code WHERE region_code = 5;
```

However, if the application needs all regions based on the timezone, then data would need to come from multiple partitons. Since these partitions are spead out over multile nodes, talking to these nodes will be expensive and can cause performance issues on a large cluster.

Cassandra will not run this query because it tries to avoid expensive queries that may cause performance issues:

```sql
SELECT * FROM geo_data.regions_by_code WHERE timezone = 'Europe/Paris';
```
```shell
InvalidRequest: Error from server: code=2200 [Invalid query] message="Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the performance unpredictability, use ALLOW FILTERING"
```

Since we want to filter by a column that is not a partition key (i.e. `timezone`), we have to tell Cassandra to filter by a non-partition key column using `ALLOW FILTERING`.

```sql
SELECT * FROM geo_data.regions_by_code WHERE timezone = 'Europe/Paris' ALLOW FILTERING;
```
```shell
 region_code | country | region | region_name | timezone
-------------+---------+--------+-------------+--------------
       34980 |     fra |    nor |   normandie | Europe/Paris
       34974 |     fra |    bre |    bretagne | Europe/Paris
```

Performing queries without conditions, such as those lacking a WHERE clause, or with conditions that do not include the partition key, can incur significant overhead and should be minimized to avoid potential performance bottlenecks.

Now the question is: how can we get all the rows from the table in a way that is performant and scalable?

The answer has been discusssed already :).

**Choose a partition key that minimizes the number of partitions accessed during read operations while evenly distributing write operations across the cluster.**

## Replication

Partitioning is certainly useful but it will not be enough to achieve high scalability. Let's examine why this is the case.

1. Single partition key space: Each partition key maps to a specific node in the cluster, and all data with the same partition key resides on that node. As the amount of data grows, the capacity of individual nodes can become a bottleneck. Eventually, the storage capacity, memory, or compute resources of a single node may be insufficient to handle the increasing volume of data or workload.
2. Hot spots: In some cases, certain partition keys may become "hot spots," meaning they are accessed more frequently than others. Hot spots can occur due to uneven data distribution or skewed access patterns, leading to uneven load distribution across nodes. Hot spots can impact performance and scalability by overloading specific nodes while underutilizing others.
3. Limited horizontal scaling: While partitioning allows Cassandra to scale horizontally by adding more nodes to the cluster, it does not address all scalability challenges. Adding more nodes can increase the cluster's overall capacity and throughput, but it does not necessarily improve performance for specific partitions or hot spots. In some cases, adding more nodes may even worsen the hot spot issues if the new nodes do not effectively distribute the workload.
4. Data skew: In real-world applications, data distribution may not be perfectly uniform, leading to data skew. Data skew occurs when certain partitions or partition keys contain significantly more data than others. Data skew can lead to uneven resource utilization, hot spots, and performance degradation.

To overcome the limitations of scalability using partitioning alone, Cassandra employs additional mechanisms such as data replication.

By replicating data to different nodes, we can access more data simultaneously from other nodes to improve latency and throughput. Replication also enables the cluster to service read and write operations in case a replica is not available.

It's a requirement to define a replication factor for every keyspace in Cassandra. In production, replication factor is typically set to 3, but other values are also possible. 

| Replication Factor | Affect                                                                                                                                                                                                                                                                                            |
|--------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1                  | Only a single copy of each row exists in the cluster. If the node containing the row goes down, the row cannot be retrieved.                                                                                                                                                                      |
| 2                  | Two copies of each row exist, where each copy is on a different node.  All replicas are equally important; there is no primary replica.                                                                                                                                                           |
| 3                  | Each row is replicated across 3 different nodes ensuring high availability. The node the receives the write request becomes the coordinator node,  it is responsible for replicating that data to two other nodes. Provides a good balance between fault tolerance, consistency, and performance. |

As a general guideline, the replication factor should ideally be less than or equal to the number of nodes in the cluster. However, there is flexibility to increase the replication factor initially and then scale the cluster by adding additional nodes as needed.

What happens when a node fails during a read operation? Rather than reading from one of the replica nodes, Cassandra sends the request to multiple replicas and picks the most updated version from the set of results it gets. Most updated values can be defined by version number or any other ID that is increasing monotonically.

## Consistency Level

Cassandra supports a tunable consistency model which provides the ability to adjust the level of consistency for read and write operations based on specific requirements and trade-offs. Cassandra provides tunable consistency levels to accommodate a wide range of use cases, balancing consistency, availability, and partition tolerance.

A consistency level of ONE is the default or both read and write operations. For all other available options, refer to the help [page](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/dml/dmlConfigConsistency.html).

Let's look at some of the more important consistency levels.

### Tunable Consistency
The levels of consistency and availability are adjustable to meet certain requirements. Individual read and write operations define the number of replicas that must acknowledge a request in order for that request to succeed.

| Consistency Level | Description                                                                                                                                                                                                                                                   |
|-------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ONE, TWO, THREE   | The number of replicas that must respond to a read or write operation.                                                                                                                                                                                        |
| LOCAL_ONE         | One replica in the same data center as the coordinator must successfully respond to  the read or write request. Provides low latency at the expense of consistency.                                                                                           |
| LOCAL_QUORUM      | A quorum (majority) of the replica nodes in the same data center as the coordinator must  respond to the read or write request. Avoids latency of inter-data-center communication.                                                                            |
| QUORUM            | A quorum (majority) of the replicas in the cluster need to respond to a read or write  request for it to succeed. Used to maintain strong consistency across the entire cluster.                                                                              |
| EACH_QUORUM       | The read or write must succeed on a quorum of replica nodes in each data center.  Used only for writes to a multi-data-center cluster to maintain the same level of consistency across data centers.                                                          |
| ALL               | The request must succeed on all replicas. Provides the highest consistency and the lowest availability of any other level.                                                                                                                                    |
| ANY               | A write must succeed on at least one node or, if all replicas are down, a hinted handoff has been written. Guarantees that a write will never fail at the expense of having the lowest consistency. Delivers the lowest consistency and highest availability. |

### Strong Consistency
In a strong consistency model, when a read operation returns a value, it guarantees that the value is the latest one that was successfully written to at least one replica node in the cluster.

According to the CAP theorem, the cluster can not be available and consistent at the same time when nodes can not communicate with each other. A strong consistency model implies that data consistency is favored when this happens even if it means that some nodes are marked as unavailable. When a Cassandra node becomes unavailable, processing continues and failed writes are temporarily saved as hints on the coordinator node. If the hints have not expired, they are applied to the node when it becomes available.

### Tune for Strong Consistency
Let's recall: Consistency level means how many nodes need to acknowledge a read or a write query for that operation to be considered successful.

To achieve strong consistency, the number of replica nodes that respond to a read or write operation must be greater than the replication factor.

```
R + W > N
where R = Read Consistency, W = Write Consistency, N = Replication Factor
```

#### Read Heavy System
For a read-heavy system, it's a good idea to keep read consistency low because reads vastly outnumber writes. 
Assuming a replication factor of 3:
```
1 + W > 3
```
Therefore, the write consistency must be 3 to achieve strong consistentcy in a read-heavy system.

#### Write Heavy System
For a write-heavy system, it's a good idea to keep write consistency low because writes vastly outnumber reads. 
Assuming a replication factor of 3:
```
R + 1 > 3
```
Therefore, the read consistency must be 3 to achieve strong consistentcy in a write-heavy system.

### Tune for Eventual Consistency
If strong consistency is not needed, you can reduce the consistency level for queries to 1 for higher performance:
```sql
CONSISTENCY ONE;
SELECT * FROM geo_data.regions_by_country WHERE country = 'usa';
```
```shell
Consistency level set to ONE.

 country | region_name | region | region_code | timezone
---------+-------------+--------+-------------+---------------------
     usa |  california |     ca |           5 | America/Los_Angeles
     usa |    colorado |     co |           6 |      America/Denver
     usa |    virginia |     va |          47 |    America/New_York
```

The data will eventually be spread to all replicas. This will ensure eventual consistency. How fast the consistency is achieved depends on different flows that sync data between nodes.

## Data Storage Optimizations
Cassandra is optimized for writes due to how it stores the data internally. Cassandra uses a log-structured storage design, where all writes are initially appended to an in-memory data structure called the memtable and then flushed to disk in immutable SSTable files. This sequential write pattern is highly efficient and minimizes the disk I/O required for write operations, resulting in lower latency and cost.

Reading is more expensive since it may require checking different disk locations until all the query data is eventually found.

As with consistency levels, Cassandra's storage engine can be tuned for reading performance or writing performance.

### Data Compaction
Data compaction in Cassandra refers to the process of organizing and optimizing data stored in on disk to improve performance and reduce storage overhead.

Cassandra allows you to set various merge and compaction strategies for a table. These strategies affect read and write performance:
1. Size-tiered Compaction Strategy (STCS): STCS is the default compaction strategy in Cassandra. STCS is well-suited for write-heavy workloads and can efficiently reclaim disk space by removing obsolete data.
2. Leveled Compaction Strategy (LCS): LCS is designed to minimize read amplification and improve read performance by reducing the number of SSTables that need to be scanned during read operations. LCS is often preferred for read-heavy workloads or use cases requiring predictable read latencies.
3. Time-window Compaction Strategy (TWCS): TWCS is optimized for time-series data. TWCS helps optimize query performance for time-based queries and ensures efficient data retention and expiration policies.
4. Date-tiered Compaction Strategy (DTCS): DTCS is a variation of TWCS. DTCS is well-suited for workloads with data expiration policies or retention periods and can efficiently manage time-series data with varying access patterns.

It is important to consider the compaction startegy at the outset when creating the tables. Although you can alter the compaction strategy of an existing table, it will lead to significant performance issues in a production system.

## Data Presorting
In Cassandra, data is already sorted on disk, eliminating the need for sorting it later. By implementing sorting at the table level, the necessity of sorting data within client applications querying Cassandra can be avoided.

Let's add `timezone` as another clustering column in our sample table and populate it with the same data:

```sql
CREATE TABLE geo_data.regions_by_country_sort_by_tz_asc (
    country text,
    region text,
    region_name text,
    region_code int,
    timezone text,
    PRIMARY KEY ((country), timezone, region_name)
);
```
```sql
BEGIN BATCH 

INSERT INTO geo_data.regions_by_country_sort_by_tz_asc (country, region, region_name, region_code, timezone)
  VALUES('fra','bre','bretagne',34974,'Europe/Paris');

INSERT INTO geo_data.regions_by_country_sort_by_tz_asc (country, region, region_name, region_code, timezone)
  VALUES('fra','nor','normandie',34980,'Europe/Paris');

INSERT INTO geo_data.regions_by_country_sort_by_tz_asc (country, region, region_name, region_code, timezone)
  VALUES('usa','ca','california',5,'America/Los_Angeles');

INSERT INTO geo_data.regions_by_country_sort_by_tz_asc (country, region, region_name, region_code, timezone)
  VALUES('usa','co','colorado',6,'America/Denver');

INSERT INTO geo_data.regions_by_country_sort_by_tz_asc (country, region, region_name, region_code, timezone)
  VALUES('usa','va','virginia',47,'America/New_York');

APPLY BATCH;
```

```sql
SELECT * FROM geo_data.regions_by_country_sort_by_tz_asc where country = 'usa';
```
```shell
 country | timezone            | region_name | region | region_code
---------+---------------------+-------------+--------+-------------
     usa |      America/Denver |    colorado |     co |           6
     usa | America/Los_Angeles |  california |     ca |           5
     usa |    America/New_York |    virginia |     va |          47
```

In this table, keep in mind that the clustering columns are `timezone` and `region_name`
From the results, we see that Cassandra sorted the data first by `timezone` and then by `region_name`.

At its core, Cassandra is still like a key-value store. Therefore, you can only query the table by:
* `country`
* `country` and `timezone`
* `country`, `timezone`, and `region_name`

You can not query the table by:
* `country` and `region_name`
* `country` and `region`
* `country` and `region_code`

## Data Modeling
Let us now create a data model for an application that allows users to create grocery lists.

Since table design is dictated by query access patterns, we need to analyze the usage with user stories. For brevity, we'll just consider a few scenarios. 

User Story 1: As a user, I want to add items to a grocery list.

User Story 2: As a user, I want to see all of the items in my grocery list by insertion time (i.e., newest elements first).

A query pattern emerges after examining these user stories. It's obvious that we need a table to store and retrieve grocery list items by user id. This ensures that all item data for a particular user id are stored on the same partition. Also, user wants to see the items sorted by time which means we'll need a `created_at` column that will also serve as a clustering key.

Let's create a keyspace and the initial table:
```sql
CREATE KEYSPACE grocery_list 
  WITH REPLICATION = { 
   'class' : 'NetworkTopologyStrategy',
   'datacenter1' : 3 
};
```
```sql
CREATE TABLE grocery_list.items_by_user_id (
    user_id int,
    name text,
    created_at timestamp,
    PRIMARY KEY ((user_id), created_at)
) WITH CLUSTERING ORDER BY (created_at DESC)
AND compaction = { 'class' :  'LeveledCompactionStrategy'  };
```
