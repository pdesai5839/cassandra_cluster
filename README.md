# Cassandra Cluster Playground

This project will demo several key concepts in Cassandra NoSQL database such as replication and tunble consistency level.
Good news is that everything can be done locally on your Mac.

## Install Cassandra

```brew install cassandra```

By default, Cassandra uses port 7000 for cluster communication (port 7001 if SSL is enabled), port 9042 for native protocol clients, and port 7199 for JMX.

Let's use Docker to fire up a 3 node cluster using Cassandra 3.11.

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
