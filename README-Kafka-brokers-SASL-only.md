# Installing and setting up a Kafka (broker) cluster with SASL

## Introduction

Instructions for setting up a Kafka cluster and enabling SASL authentication between the brokers and ZooKeeper follow.

My setup will be as follows:

- 2 nodes, each with just one broker. I was going to have a node with two brokers, but then I figured... why? There does not seem to be any advantage in such an arrangement.
- The `/chroot/path` will be `/apps/kafka-cluster-demo`. This will be a znode in the ZooKeeper cluster that I will create manually. I will give `cdrw` permissions to the brokers for this znode. This is like giving a namespace for my Kafka cluster, so that multiple applications can use the same ZooKeeper ensemble without interfering with each other.  

## Steps

### Setting up the first Kafka node.

Instructions for setting up the server first Kafka node follow. After you have set up the server for the first Kafka node, you can follow these same steps for setting up the servers for the other nodes in the cluster, with the exception that you would use a different static IP address for each and change the subdomain from `kerberos-server-01` to `server-k`, where `k` is `02`, `03`, etc. (Here, we will assume that the Kafka nodes are on servers that could be running other applications, so we will call them `server-k` instead of `kafka-server-k`).

Steps 1 through 7, are the same as [those for setting up Kerberos](README-Kerberos.md), except that for this demonstration the subdomain will be `server-01` (so you will replace the contents of `/etc/hostname` with `server-01.mydomain.com` and add `YOUR.STATIC.IP.ADDRESS server-01` and `YOUR.STATIC.IP.ADDRESS server-01.yourdomain.com` to `/etc/hosts`). Also, make sure that the following ports are open: 9092, 9093. If you will be running multiple brokers on a server, then open 2 additional ports for each additional broker.

At this point you should have your Kafka servers set up. What makes a Kafka server a Kafka node is running at least one Kafka broker on the server. Below are instructions for setting up broker-10.

8. In your ZooKeeper ensemble, create a znode to be used as the `/chroot/path` for this demo. I will use `/apps/kafka-cluster-demo` as my `/chroot/path` You can connect using the `zktestclient` from the [ZooKeeper setup instructions](README-ZooKeeper.md). Also set the permissions so that all brokers

    ```
    create /apps
    setAcl /apps sasl:zktestclient/whatever@YOURDOMAIN.COM:crwd
    create /apps/kafka-cluster-demo ""
    setAcl /apps/kafka-cluster-demo sasl:zktestclient/whatever@YOURDOMAIN.COM:crwda,sasl:kafka/server-01.yourdomain.com@YOURDOMAIN.COM:crw,sasl:kafka/server-02.yourdomain.com@YOURDOMAIN.COM:crw
    ```

9. Create a `kafka` user: `sudo adduser kafka`, and choose a password.

10. `su` to this user, and [download](https://kafka.apache.org/downloads) Kafka 1.0 (don't automatically use the mirror below - choose the mirror that the above link suggests for you):

    ```
    sudo su - kafka
    mkdir ~/log
    wget -c http://apache.mirror.globo.tech/kafka/1.0.0/kafka_2.11-1.0.0.tgz
    tar -zxvf kafka_2.11-1.0.0.tgz
    exit
    sudo ln -s /home/kafka/kafka_2.11-1.0.0 /opt/kafka
    sudo su - kafka
    cd kafka_2.11-1.0.0
    ```

11. Create a copy of the `server.properties` file (in the `config/` directory) for this exercise:

    ```
    cp config/server.properties config/server-sasl-brokers-zookeeper.properties
    ```

    Edit the new file so that it contains the following (add or edit existing lines):

    ```
    broker.id=1 # for other brokers, replace the 1 with SERVER_NUMBER (which is the same as NODE_NUMBER, since we will have only one node per server).
    listeners=SASL_PLAINTEXT://0.0.0.0:9092
    advertised.listeners=SASL_PLAINTEXT://server-01.yourdomain.com:9092 # change server-01 as appropriate
    security.inter.broker.protocol=SASL_PLAINTEXT
    super.users=kafka
    delete.topic.enable=true
    auto.create.topics.enable
    log.dirs=/home/kafka/log
    zookeeper.connect=zookeeper-server-01.yourdomain.com:2181,zookeeper-server-02.yourdomain.com:2181,zookeeper-server-03.yourdomain.com:2181/apps/kafka-cluster-demo # Include all of your ZooKeeper servers - I use 3.
    ```

    A sample server config file is available [here](server-sasl-brokers-zookeeper.properties)

12. Create a `jaas.conf` file in the `config/` directory:

    ```
    KafkaServer {
      com.sun.security.auth.module.Krb5LoginModule required
      useKeyTab=true
      keyTab="/home/kafka/kafka_2.11-1.0.0/config/kafka.server-01.yourdomain.com.keytab"
      storeKey=true
      useTicketCache=false
      serviceName="kafka"
      principal="kafka/server-01.yourdomain.com@YOURDOMAIN.COM";
    };

    // This is for the broker acting as a client to ZooKeeper
    Client {
      com.sun.security.auth.module.Krb5LoginModule required
      useKeyTab=true
      keyTab="/home/kafka/kafka_2.11-1.0.0/config/kafka.server-01.yourdomain.com.keytab"
      storeKey=true
      useTicketCache=false
      serviceName="zookeeper"
      principal="kafka/server-01.yourdomain.com@YOURDOMAIN.COM";
    };
    ```

13. Place the keytab file cited in the above `jaas.conf` file (that were created in [the Kerberos setup instructions](README-Kerberos.md) - `kafka.server-01.yourdomain.com.keytab`) in the `config/` directory.

14. Test that you can start the broker without any errors:

    ```
    KAFKA_HEAP_OPTS="-Djava.security.auth.login.config=/home/kafka/kafka_2.11-1.0.0/config/jaas.conf -Dsun.security.krb5.debug=true -Djava.security.krb5.conf=/etc/krb5.conf -Xmx256M -Xms128M" \
      bin/kafka-server-start.sh config/server-sasl-brokers-zookeeper.properties
    ```

    Note that I have added the JVMFLAGS `-Xmx256M` and `-Xms128M` to limit the amount of memory that Kafka will use, because I am using an AWS t2.miro instance, which has only 1 GB of RAM. If I don't add these flags, I will get an out of memory error.

15. If the previous step succeeds, then test that you can write to a Kafka topic on this broker with a producer, and read from it using a consumer. Make sure that the broker you started from the step above is still running.

    a. Open a separate terminal window, log in as Kafka, and switch to the `/home/kafka/kafka_2.11-1.0.0/` directory. Create a test topic to publish to:

        KAFKA_OPTS="-Djava.security.auth.login.config=/home/kafka/kafka_2.11-1.0.0/config/jaas.conf -Djava.security.krb5.conf=/etc/krb5.conf"  \
          bin/kafka-topics.sh --create \
          --zookeeper zookeeper-server-01.yourdomain.com:2181,zookeeper-server-02.yourdomain.com:2181,zookeeper-server-03.yourdomain.com:2181/apps/kafka-cluster-demo \
          --replication-factor 1 \
          --partitions 1 \
          --topic test-topic

    You should see `Created topic "test-topic".` as the output of this command.

    b. Now list all topics, just to confirm (not really necessary) that the test topic `test-topic` was created:

        KAFKA_OPTS="-Djava.security.auth.login.config=/home/kafka/kafka_2.11-1.0.0/config/jaas.conf -Djava.security.krb5.conf=/etc/krb5.conf"  \
          bin/kafka-topics.sh --list \
          --zookeeper zookeeper-server-01.yourdomain.com:2181,zookeeper-server-02.yourdomain.com:2181,zookeeper-server-03.yourdomain.com:2181/apps/kafka-cluster-demo

    The output should be `test-topic`. You can get detail on the topic by executing the above command using `--describe` instead of `--list`.

    c. At this point, attempts to write to or read from this topic will fail, because no producers and no consumers have authorizations to do so. Lets us grant these authorizations for the producer:

        KAFKA_HEAP_OPTS="-Djava.security.auth.login.config=/home/kafka/kafka_2.11-1.0.0/config/jaas.conf -Dsun.security.krb5.debug=true -Djava.security.krb5.conf=/etc/krb5.conf -Xmx256M -Xms128M" \
          bin/kafka-acls.sh --authorizer-properties \
          zookeeper.connect=zookeeper-server-01.yourdomain.com:2181,zookeeper-server-02.yourdomain.com:2181,zookeeper-server-03.yourdomain.com:2181/apps/kafka-cluster-demo \
          --add --allow-principal User:producer1/whatever@EIGENROUTE.COM --producer --topic test-topic

    and for the consumer:    

        KAFKA_HEAP_OPTS="-Djava.security.auth.login.config=/home/kafka/kafka_2.11-1.0.0/config/jaas.conf -Dsun.security.krb5.debug=true -Djava.security.krb5.conf=/etc/krb5.conf -Xmx256M -Xms128M" \
          bin/kafka-acls.sh --authorizer-properties \
          zookeeper.connect=zookeeper-server-01.yourdomain.com:2181,zookeeper-server-02.yourdomain.com:2181,zookeeper-server-03.yourdomain.com:2181/apps/kafka-cluster-demo \
          --add --allow-principal User:consumer1/whatever@EIGENROUTE.COM --consumer --topic test-topic --group Group-1

### Testing that you can write to and read from a topic

16. Now we will configure and start producers and consumers on a separate machine. You can use your local machine. Navigate to or create a directory for testing the consumer and producer.

    ```
    cd /path/to/your-directory/
    wget -c http://apache.mirror.globo.tech/kafka/1.0.0/kafka_2.11-1.0.0.tgz
    tar -zxvf kafka_2.11-1.0.0.tgz
    rm kafka_2.11-1.0.0.tgz
    cd kafka_2.11-1.0.0
    ```

    Place the `producer1.whatever.keytab` and `consumer.whatever.keytab` files you created [earlier](README-kerberos.md) in the `config/` directory. In this directory, create a file `sasl-producer.properties` with the following contents:

    ```
    bootstrap.servers=server-01.yourdomain.com:9092
    security.protocol=SASL_PLAINTEXT
    sasl.mechanism=GSSAPI
    sasl.kerberos.service.name=kafka
    sasl.jaas.config=com.sun.security.auth.module.Krb5LoginModule required \
      useKeyTab=true \
      storeKey=true  \
      keyTab="/path/to/producer1.whatever.keytab" \
      principal="producer1/whatever@YOURDOMAIN.COM";
    ```

    And create a file `sasl-consumer.properties` with the following contents:

    ```
    bootstrap.servers=server-01.yourdomain.com:9092
    security.protocol=SASL_PLAINTEXT
    sasl.mechanism=GSSAPI
    sasl.kerberos.service.name=kafka
    sasl.jaas.config=com.sun.security.auth.module.Krb5LoginModule required \
      useKeyTab=true \
      storeKey=true  \
      keyTab="/path/to/consumer1.whatever.keytab" \
      principal="consumer1/whatever@YOURDOMAIN.COM";
    ```

17. In this same terminal window, start the producer:

    ```
    KAFA_HEAP_OPTS="-Djava.security.krb5.conf=/etc/krb5.conf -Dsun.security.krb5.debug=true" \
      bin/kafka-console-producer.sh --broker-list server-01.yourdomain.com:9092 \
      --topic test-topic --producer.config config/sasl-producer.properties
    ```

    In a different terminal, navigate to the same directory that you were in above, and start the consumer:

    ```
    KAFA_HEAP_OPTS="-Djava.security.krb5.conf=/etc/krb5.conf -Dsun.security.krb5.debug=true" \
      bin/kafka-console-consumer.sh --bootstrap-server server-01.yourdomain.com:9092 \
      --topic test-topic --consumer.config config/sasl-consumer.properties --from-beginning
    ```

    Start entering text in the producer window, hit enter, and check that this same text shows up in the consumer window. If so, your single node cluster works, and you can now stop the producer and the consumer. If not, open an issue in this repo, so that I can fix these instructions.

18. Delete this test topic:

    ```
    sudo su - kafka
    cd /home/kafka/kafka_2.11-1/
    KAFKA_HEAP_OPTS="-Djava.security.auth.login.config=/home/kafka/kafka_2.11-1.0.0/config/jaas.conf -Dsun.security.krb5.debug=true -Djava.security.krb5.conf=/etc/krb5.conf -Xmx256M -Xms128M" \
      bin/kafka-run-class.sh kafka.admin.TopicCommand \
      --zookeeper zookeeper-server-01.yourdomain.com:2181,zookeeper-server-02.yourdomain.com:2181,zookeeper-server-03.yourdomain.com:2181/apps/kafka-cluster-demo \
      --delete --topic test-topic
    ```    

    Note: deleting topics is flaky. You might have to connect to Zookeeper as `super` and delete the topic manually by executing the following:

    ```
    rmr /apps/kafka-cluster-demo/brokers/topics/test-topic
    ```

    (the reference for this is [here](https://stackoverflow.com/a/33538299/2251463))

### Make the Kafka broker a service

19. If the previous step succeeds, then make this Kafka broker a service using the `init.d` script [here](kafka). Go back to the windown where your Kafka broker is running, and stop the broker. Place this script in the `/etc/init.d/` directory, make it world executable, make it a service:

    ```
    exit # to get you back to the admin user, which can sudo
    cd /etc/init.d
    sudo chmod a+x kafka
    sudo update-rc.d kakfa defaults
    sudo service kafka start   
    ```

### Setting up the second Kafka node/broker

20. Set up a second Kafka server, just as you set up the first Kafka server - repeat steps 1 though 7, 9 through 14, and step 19 above, using a different broker ID and subdomain. I will use a broker ID of `2` and the subdomain `server-02`.

21. Now, log into the server of the first broker (server-01) as kafka, navigate to the `kafka_2.11-1.0.0/` directory, and create a topic. Basically, execute step 15 above again, except name the new test topic `test-topic2`. `describe` the new topic, and grant the appropriate permissions to the producer and consumer for the new topic.

    ```
    KAFKA_OPTS="-Djava.security.auth.login.config=/home/kafka/kafka_2.11-1.0.0/config/jaas.conf -Djava.security.krb5.conf=/etc/krb5.conf"  \
      bin/kafka-topics.sh --create \
      --zookeeper zookeeper-server-01.yourdomain.com:2181,zookeeper-server-02.yourdomain.com:2181,zookeeper-server-03.yourdomain.com:2181/apps/kafka-cluster-demo \
      --replication-factor 2 \
      --partitions 9 \
      --topic test-topic2

    KAFKA_OPTS="-Djava.security.auth.login.config=/home/kafka/kafka_2.11-1.0.0/config/jaas.conf -Djava.security.krb5.conf=/etc/krb5.conf"  \
      bin/kafka-topics.sh --describe \
      --zookeeper zookeeper-server-01.yourdomain.com:2181,zookeeper-server-02.yourdomain.com:2181,zookeeper-server-03.yourdomain.com:2181/apps/kafka-cluster-demo
   ```    

    The output should show that `test-topic2` has 9 partitions, that the leader of each partition is either 1 or 2, and that the Isrs are either 1,2 or 2,1. If instead you get output showing that, for all partitions, the leader is "none" and the Isrs are blank, then resart both brokers. I found this to be necessary if I create the topic on Broker 2 instead of Broker 1. See [here](https://stackoverflow.com/questions/48143483/kafka-producer-in-a-multi-broker-multi-server-cluster-cannot-write-to-newly-cre) for more info. Weird.

    Now grant the necessary permissions for the producer and the consumer:  

    ```
    KAFKA_HEAP_OPTS="-Djava.security.auth.login.config=/home/kafka/kafka_2.11-1.0.0/config/jaas.conf -Dsun.security.krb5.debug=true -Djava.security.krb5.conf=/etc/krb5.conf -Xmx256M -Xms128M" \
      bin/kafka-acls.sh --authorizer-properties \
      zookeeper.connect=zookeeper-server-01.yourdomain.com:2181,zookeeper-server-02.yourdomain.com:2181,zookeeper-server-03.yourdomain.com:2181/apps/kafka-cluster-demo \
      --add --allow-principal User:producer1/whatever@EIGENROUTE.COM --producer --topic test-topic2

    KAFKA_HEAP_OPTS="-Djava.security.auth.login.config=/home/kafka/kafka_2.11-1.0.0/config/jaas.conf -Dsun.security.krb5.debug=true -Djava.security.krb5.conf=/etc/krb5.conf -Xmx256M -Xms128M" \
      bin/kafka-acls.sh --authorizer-properties \
      zookeeper.connect=zookeeper-server-01.yourdomain.com:2181,zookeeper-server-02.yourdomain.com:2181,zookeeper-server-03.yourdomain.com:2181/apps/kafka-cluster-demo \
      --add --allow-principal User:consumer1/whatever@EIGENROUTE.COM --consumer --topic test-topic2 --group Group-1
    ```  

    This also might not work on the broker server on which you are executing the command, as you might have to execute the command on the other broker. Yes, more Kafka weirdness. For me, the producer command failed on the Broker 1 server with the following error:

    ```
    Error while executing ACL command: org.apache.zookeeper.KeeperException$NoAuthException:
      KeeperErrorCode = NoAuth for /kafka-acl-changes/acl_changes_0000000000
org.I0Itec.zkclient.exception.ZkException: org.apache.zookeeper.KeeperException$NoAuthException:
  KeeperErrorCode = NoAuth for /kafka-acl-changes/acl_changes_0000000000

    ```

    When I checked the permissions on ZooKeeper of the `/apps/kafka-cluster-demo/kafka-acl-changes/acl_changes_0000000000` node, I saw that only Broker 2 has write permission on this node:

    ```
    [zk: zookeeper-server-03.eigenroute.com:2181(CONNECTED) 0] getAcl /apps/kafka-cluster-demo/kafka-acl-changes/acl_changes_0000000000
    'world,'anyone
    : r
    'sasl,'kafka/server-02.eigenroute.com@EIGENROUTE.COM
    : cdrwa
    [zk: zookeeper-server-03.eigenroute.com:2181(CONNECTED) 1]
    ```

    So I had to set the permissions using the Broker 2 server. This worked - the producer was able to write to and the consumer was able to read from the topic.

22. On your local machine, as before, start a producer and consumer in separate terminal tabs. Use the same producer and consumers and before, except set the topic to `test-topic2` for both. Enter text in the producer tab and you should see it output in the consumer tab.


Well this was a pain. I found two instances of Kafka weirdness. It shouldn't matter on which Broker server you create topics or grant permissions, but apparently it does.

Next up: SSL between Kafka clients and Brokers.     


## References

Kumar, Manish; Chanchal Singh. Building Data Streaming Applications with Apache Kafka. Packt Publishing. August 2017. Publication reference 1170817.

Estrada, Raul. Apache Kafka 1.0 Cookbook. Packt Publishing. December 2017. Publication reference 1211217.

https://www.confluent.io/blog/apache-kafka-security-authorization-authentication-encryption/

https://www.gitbook.com/book/jaceklaskowski/apache-kafka/details

https://kafka.apache.org/

https://stackoverflow.com/a/33538299/2251463

https://stackoverflow.com/questions/48143483/kafka-producer-in-a-multi-broker-multi-server-cluster-cannot-write-to-newly-cre
