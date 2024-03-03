# Cassandra Cluster Playground

This project will demo several key concepts in Cassandra NoSQL database such as replication and tunble consistency level.

Good news is that everything can be done locally on your Mac.

## Install Cassandra

`brew install cassandra`

By default, Cassandra uses port 7000 for cluster communication (port 7001 if SSL is enabled), port 9042 for native protocol clients, and port 7199 for JMX.

Let's use Docker to fire up a 3 node cluster using Cassandra 3.11. Note that this could take a while.

```
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

It should output something similar to this:
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

```
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.16 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh>
```

## Keyspaces

Time to execute our first query. Run the `DESCRIBE` command:
```
cqlsh> DESCRIBE keyspaces;

system_traces  system_schema  system_auth  system  system_distributed
```

The `DESCRIBE keyspaces;` command is used to display information about the keyspaces present in the Cassandra database cluster.

So what are keyspaces anyway? Let's define it in simple terms:
Keyspaces are the highest-level containers for organizing data. Think of the more familiar Schema/Database in traditional SQL RDMS and you'll get a feel for what Keyspaces are. Keyspaces are top-level containers for organizing data. However, this is where the similarity ends.

So knowing what keyspaces are, we must create the keyspaces first in order to create tables. We'll use passenger arrivals/departues traffic for San Francisco International Airport to play around with Cassandra.

```
CREATE KEYSPACE sfo_passenger_traffic 
  WITH REPLICATION = { 
   'class' : 'NetworkTopologyStrategy',
   'datacenter1' : 3 
  };
```

What did we just do? We just created a keyspace named "sfo_passenger_traffic" with these replication settings:
* The replication strategy specified is "NetworkTopologyStrategy". This strategy is used to replicate data across multiple data centers in a Cassandra cluster.
* Within the "NetworkTopologyStrategy", the replication factor for the "datacenter1" is set to 3. This means that the data in this keyspace will be replicated across three nodes within the "datacenter1" data center.

# Importance of Replication Factor

In Cassandra, setting a replication factor of 3 is a common practice for several reasons:
* *Fault Tolerance*: Cassandra is designed to be highly fault-tolerant. By replicating data across multiple nodes (in this case, three nodes), it ensures that there are multiple copies of each piece of data available. If one node goes down or becomes unavailable, the data can still be accessed from the other replicas, ensuring high availability.
* *Consistency*: Cassandra uses a distributed consensus protocol to ensure data consistency. With a replication factor of 3, Cassandra can achieve strong consistency guarantees while still providing high availability. If one node becomes unavailable, Cassandra can use the remaining replicas to maintain data consistency.
* *Read and Write Performance*: Replicating data across multiple nodes can improve read and write performance. With multiple replicas available, Cassandra can distribute read and write requests across these replicas, reducing the load on individual nodes and improving overall system performance.
* *Data Distribution*: Cassandra uses a decentralized architecture where data is distributed across multiple nodes in the cluster. A replication factor of 3 helps ensure that data is evenly distributed across the cluster, preventing hotspots and ensuring efficient data access.

