# Installing and Setting up a Zookeeper ensemble with SASL, using Debian Stretch (Debian 9.2)

## Introduction

Instructions for setting up a Zookeper ensemble and enabling SASL follow.

//TODO: This is flaky - still need to get this part working

## Steps

### Setting up the first Zookeeper server.

Instructions for setting up the first Zookeeper server follow. After you have set up the first Zookeper server, you can follow these same steps for setting up the other servers in the ensemble, with the exception that you would use a different static IP address and change the subdomain from `zookeeper-server-01` to `zookeeper-server-k`, where k is `02`, `03`, etc.

Steps 1 through 6, are the same as those for setting up Kerberos, except that for this demonstration the subdomain will be `zookeeper-server-01` (so you will replace the contents of `/etc/hostname` with `zookeeper-server-01.mydomain.com` and add `YOUR.STATIC.IP.ADDRESS zookeeper-server-01` to `/etc/hosts`).

7. Add a user zookeeper:

   `sudo adduser zookeeper`  

8. Switch to this user, and download and unpack the latest stable version of zookeeper (don't automatically use the link below - use a download link available from [here](https://www.apache.org/dyn/closer.cgi/zookeeper/)). I'm using 3.4.11, which seems to be the latest stable version as of the time of this post:

    ```
    sudo su - zookeeper
    mkdir zk
    cd zk
    wget -c http://apache.mirror.globo.tech/zookeeper/zookeeper-3.4.11/zookeeper-3.4.11.tar.gz
    tar -zxvf zookeeper-3.4.11.tar.gz
    rm zookeeper-3.4.11.tar.gz
    cd zookeeper-3.4.11
    ```

9. In the `conf/` directory, create a file `zoo.cfg-ensemble` with the following contents:

    ```
    tickTime=2000
    initLimit=10
    dataDir=/home/zookeeper/dataDir
    dataLogDir=/home/zookeeper/dataLogDir
    clientPort=2181
    server.1=zookeeper-server-01.yourdomain.com:2888:3888
    server.2=zookeeper-server-02.yourdomain.com:2888:3888
    server.3=zookeeper-server-03.yourdomain.com:2888:3888
    authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
    requireClientAuthScheme=sasl
    jaasLoginRenew=3600000
    ```       
10. Create the `dataDir` and `dataLogDir` folders, and a `jaas folder`, and create a symlink from `zoo.cfg` to `zoo.cfg-ensemble`:

    ```
    mkdir /home/zookeeper/dataDir
    mkdir /home/zookeeper/dataLogDir
    mkdir /home/zookeeper/log
    mkdir /home/zookeeper/jaas
    ln -s /home/zookeeper/zk/zookeeper-3.4.11/conf/zoo.cfg-ensemble /home/zookeeper/zk/zookeeper-3.4.11/conf/zoo.cfg
    ```
11. In the `jaas/` folder, place the `zookeeper-server-01.keytab` file that you created when setting up Kerberos. Also in this folder, create a `jaas.conf` file with the following contents:

    ```
    Server {
            com.sun.security.auth.module.Krb5LoginModule required
            useTicketCache=false
            useKeyTab=true
            keyTab="/home/zookeeper/jaas/zookeeper-server-01.keytab"
            storeKey=true
            principal="zookeeper/zookeeper-server-01@YOURDOMAIN.COM";
    };
    ```

12. In `/home/zookeeper/dataDir/`, create a file `myid` and place a solitary `1` in this file:

    ```
    echo "1" > /home/zookeeper/dataDir/myid
    ```

`1` is used here because this is `zookeeper-server-01`. For `zookeeper-server-02`, one would use a `2` instead, etc.

13. Repeat the above steps for the other ZooKeeper servers in the ensemble (`zookeeper-server-02` and `zookeeper-server-03`)


## References



Junqueira, Flavio; Reed, Benjamin. ZooKeeper: Distributed Process Coordination (Kindle Location 743). O'Reilly Media. Kindle Edition.
