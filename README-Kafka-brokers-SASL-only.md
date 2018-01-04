# Installing and setting up a Kafka (broker) cluster with SASL

## Introduction

Instructions for setting up a Kafka cluster and enabling SASL authentication between the brokers and ZooKeeper follow.

My setup will be as follows:

- 2 nodes (each with the id of broker-X, where X is the broker id, as below)
  - node 1 will have 1 broker: broker-10
  - node 2 will have 2 brokers: broker-20, broker-21
- The `/chroot/path` will be `/apps/kafka-cluster-demo`. This will be a znode in the ZooKeeper cluster that I will create manually. I will give `cdrw` permissions to the brokers for this znode. This is like giving a namespace for my Kafka cluster, so that multiple applications can use the same ZooKeeper ensemble without interfering with each other.  

## Steps

### Setting up the first Kafka node.

Instructions for setting up the server first Kafka node follow. After you have set up the server for the first Kafka node, you can follow these same steps for setting up the servers for the other nodes in the cluster, with the exception that you would use a different static IP address for each and change the subdomain from `kerberos-server-01` to `node-k`, where `k` is `02`, `03`, etc. (Here, we will assume that the Kafka nodes are on servers that could be running other applications, so we will call them `node-k` instead of `kafka-node-k`).

Steps 1 through 7, are the same as [those for setting up Kerberos](README-Kerberos.md), except that for this demonstration the subdomain will be `node-01` (so you will replace the contents of `/etc/hostname` with `kafka-node-01.mydomain.com` and add `YOUR.STATIC.IP.ADDRESS node-01` to `/etc/hosts`). Also, make sure that the following ports are open: 9092, 9093. If you will be running multiple brokers on a server, then open 2 additional ports for each additional broker.

At this point you should have your Kafka servers set up. What makes a Kafka server a Kafka node is running at least one Kafka broker on the server. Below are instructions for setting up broker-10.

8. In your ZooKeeper ensemble, create a znode to be used as the `/chroot/path` for this demo. I will use `/apps/kafka-cluster-demo` as my `/chroot/path` You can connect using the `zktestclient` from the [ZooKeeper setup instructions](README-ZooKeeper.md). Also set the permissions so that all brokers

    ```
    create /apps
    setAcl /apps sasl:zktestclient/whatever@YOURDOMAIN.COM:crwd
    create /apps/kafka-cluster-demo ""
    setAcl /apps/kafka-cluster-demo sasl:kafka-broker-10/kafka-node-01.yourdomain.com@YOURDOMAIN.COM:cdrw,sasl:kafka-broker-20/kafka-node-02.yourdomain.com@YOURDOMAIN.COM:cdrw,sasl:kafka-broker-21/kafka-node-02.yourdomain.com@YOURDOMAIN.COM:cdrw
    ```

9. Create a `kafka` user: `sudo adduser kafka`, and choose a password.

10. `su` to this user, and [download](https://kafka.apache.org/downloads) Kafka 1.0 (don't automatically use the mirror below - choose the mirror that the above link suggests for you):

    ```
    sudo su - kafka
    wget -c http://apache.mirror.globo.tech/kafka/1.0.0/kafka_2.11-1.0.0.tgz
    tar -zxvf kafka_2.11-1.0.0.tgz
    exit
    sudo ln -s /home/kafka/kafka_2.11-1.0.0 /opt/kafka
    sudo mkdir /var/log/kafka
    sudo chown kafka. /var/log/kafka
    sudo su - kafka
    cd kafka_2.11-1.0.0
    ```

11. Create a copy of the `server.properties` file (in the `config/` directory) for this exercise:

    ```
    cp config/server.properties config/server-sasl-brokers-zookeeper.properties
    ```

    Edit the new file so that it contains the following (add or edit existing lines):

    ```
    broker.id=11 # for other brokers, replace the 11 with {NODE_NUMBER}{BROKER_NUMBER}.
    zookeeper.connect=zookeeper-server-01.yourdomain.com:2181,zookeeper-server-02.yourdomain.com:2181,zookeeper-server-03.yourdomain.com:2181/apps/kafka-cluster-demo # Include all of your ZooKeeper servers - I use 3.
    listeners=SASL_PLAINTEXT://server-01.yourdomain.com:9092 # change server-01 as appropriate
    security.inter.broker.protocol=SASL_PLAINTEXT
    sasl.kerberos.service.name=kafka-broker-1-1
    log.dirs=/var/log/kafka
    ```

12. Create a `jaas.conf` file in the `config/` directory:

    ```
    KafkaServer {
      com.sun.security.auth.module.Krb5LoginModule required
      useKeyTab=true
      keyTab="/home/kafka/kafka_2.11-1.0.0/config/kafka-broker-1-1.server-01.yourdomain.com.keytab"
      storeKey=true
      useTicketCache=false
      principal="kafka-broker-1-1/server-01.yourdomain.com@YOURDOMAIN.COM";
    };

    // This is for the broker acting as a client to ZooKeeper
    Client {
      com.sun.security.auth.module.Krb5LoginModule required
      useKeyTab=true
      keyTab="/home/kafka/kafka_2.11-1.0.0/config/kafka-broker-1-1.server-01.yourdomain.com.keytab"
      storeKey=true
      useTicketCache=false
      principal="kafka-broker-1-1/server-01.yourdomain.com@YOURDOMAIN.COM";
    };
    ```

13. Place the keytab file cited in the above `jaas.conf` file (that were created in [the Kerberos setup instructions](README-Kerberos.md) - `kafka-broker-1-1.server-01.yourdomain.com.keytab`) in the `config/` directory.

14. TBC


## References

https://kafka.apache.org/downloads
