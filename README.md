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


