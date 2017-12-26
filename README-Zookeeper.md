# Installing and Setting up a Zookeeper ensemble with SASL, using Debian Stretch (Debian 9.2)

## Introduction

Instructions for setting up a ZooKeeper ensemble and enabling SASL follow.


## Steps

### Setting up the first Zookeeper server.

Instructions for setting up the first Zookeeper server follow. After you have set up the first Zookeper server, you can follow these same steps for setting up the other servers in the ensemble, with the exception that you would use a different static IP address and change the subdomain from `zookeeper-server-01` to `zookeeper-server-k`, where `k` is `02`, `03`, etc.

Steps 1 through 7, are the same as [those for setting up Kerberos](README-Kerberos.md), except that for this demonstration the subdomain will be `zookeeper-server-01` (so you will replace the contents of `/etc/hostname` with `zookeeper-server-01.mydomain.com` and add `YOUR.STATIC.IP.ADDRESS zookeeper-server-01` to `/etc/hosts`).

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
    syncLimit=5
    dataDir=/home/zookeeper/dataDir
    dataLogDir=/home/zookeeper/dataLogDir
    clientPort=2181
    # server.1=zookeeper-server-02.yourdomain.com:2888:3888
    # server.2=zookeeper-server-02.yourdomain.com:2888:3888
    # server.3=zookeeper-server-03.yourdomain.com:2888:3888
    authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
    requireClientAuthScheme=sasl
    jaasLoginRenew=3600000
    ```

    Note: For now, the `server.X` lines are commented out, in order to test the ZooKeeper server in StandAlone mode. Later we will uncomment these lines so that we can use ZooKeeper as an ensemble of servers, at which point this file will be exactly the same for each ZooKeeper server in the ZooKeeper ensemble.

10. Create the `dataDir` and `dataLogDir` folders, and a `jaas folder`, and create a symlink from `zoo.cfg` to `zoo.cfg-ensemble`:

    ```
    mkdir /home/zookeeper/dataDir
    mkdir /home/zookeeper/dataLogDir
    mkdir /home/zookeeper/log
    mkdir /home/zookeeper/jaas
    ln -s /home/zookeeper/zk/zookeeper-3.4.11/conf/zoo.cfg-ensemble /home/zookeeper/zk/zookeeper-3.4.11/conf/zoo.cfg
    ```
11. In the `/home/zookeeper/jaas/` folder, place the `zookeeper-server-01.keytab` file that you created when setting up Kerberos. Make the `zookeeper` user the owner of this file, and make sure this user has read and write permissions on the file. Also in this folder, create a `jaas.conf` file with the following contents:

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

    Note that you would use `02` or `03` instead of `01` in the above file according to the ensemble number (id) that you are setting up.

12. In `/home/zookeeper/dataDir/`, create a file `myid` and place a solitary `1` in this file:

    ```
    echo "1" > /home/zookeeper/dataDir/myid
    ```

    `1` is used here because this is `zookeeper-server-01`. For `zookeeper-server-02`, one would use a `2` instead, etc.


13. In the `/etc/` folder, place the `/etc/krb5.conf` file from [the Kerberos server that you set up earlier](README-Kerberos.md), and make sure it is world readable.

14. Repeat the above steps for the other ZooKeeper servers in the ensemble (`zookeeper-server-02` and `zookeeper-server-03`)

15. To test that the server starts without errors, execute the following command from `/home/zookeeper/zk/zookeeper-3.4.11/`:

    ```
    JVMFLAGS="-Djava.security.auth.login.config=/home/zookeeper/jaas/jaas.conf -Dsun.security.krb5.debug=true" bin/zkServer.sh start-foreground
    ```

    The output should be similar to the following:

    ```
    ...
    >>> KrbAsRep cons in KrbAsReq.getReply zookeeper/zookeeper-server-01.eigenroute.com
    2017-12-26 20:46:20,231 [myid:] - INFO  [main:Login@297] - Server successfully logged in.
    2017-12-26 20:46:20,234 [myid:] - INFO  [main:NIOServerCnxnFactory@89] - binding to port 0.0.0.0/0.0.0.0:2181
    2017-12-26 20:46:20,235 [myid:] - INFO  [Thread-1:Login$1@130] - TGT refresh thread started.
    2017-12-26 20:46:20,235 [myid:] - INFO  [Thread-1:Login@305] - TGT valid starting at:        Tue Dec 26 20:46:20 UTC 2017
    2017-12-26 20:46:20,236 [myid:] - INFO  [Thread-1:Login@306] - TGT expires:                  Wed Dec 27 06:46:20 UTC 2017
    2017-12-26 20:46:20,236 [myid:] - INFO  [Thread-1:Login$1@185] - TGT refresh sleeping until: Wed Dec 27 04:58:21 UTC 2017
    ```    

### Testing the ZooKeeper server in StandAlone mode.

16. On another machine (EC2 instance, personal, whatever - as long as it is not the same machine as the server, though it can be), download ZooKeeper as before to a directory of your choosing - I'll assume you have downloaded it to your home directory:

    ```
    cd ~
    wget -c http://apache.mirror.globo.tech/zookeeper/zookeeper-3.4.11/zookeeper-3.4.11.tar.gz
    tar -zxvf zookeeper-3.4.11.tar.gz
    rm zookeeper-3.4.11.tar.gz
    cd zookeeper-3.4.11
    ```

17. Place the `/etc/security/zookeeperclient.whatever.keytab` that you created in the [Kerberos](README-Kerberos.md) section in some directory, say `~/jaas/` (create this directory). Make sure you have read permission on it.

18. Also in the `~/jaas/` section, create a file called `jaas.conf` and add the following content to it:

    ```
    Client {
      com.sun.security.auth.module.Krb5LoginModule required
      useTicketCache=false
      useKeyTab=true
      keyTab="~/jaas/zookeeperclient.whatever.keytab"
      storeKey=true
      principal="zookeeperclient/whatever@YOURDOMAIN.COM";
    };
    ```
19. Place the `/etc/krb5.conf` file from the Kerberos server that you set up in the `~/jaas/` directory of the client machine, and give yourself read permissions on it.

20. From the `~/zookeeper-3.4.11/` directory, run the following command:

    ```
    JVMFLAGS="-Djava.security.auth.login.config=/full/path/to/jaas/jaas.conf -Dsun.security.krb5.debug=true -Djava.security.krb5.conf=/full/path/to/jass/krb5.conf" bin/zkCli.sh -server zookeeper-server-01.eigenroute.com:2181
    ```

    If successful, the terminal output should be similar to the following:

    ```
    ...
    0290: 78 C5 E1 84 61 9E 17 78   CC 01 82 23 AB B2 5D EF  x...a..x...#..].
    02A0: 32 50 7D ED 19 EC 8E 5B   C3 5B DD 3B              2P.....[.[.;

    Krb5Context.unwrap: token=[05 04 01 ff 00 0c 00 00 00 00 00 00 2a d9 04 e7 01 01 00 00 5b 29 03 0d 76 14 1f 76 0d 1d 3e 9a ]
    Krb5Context.unwrap: data=[01 01 00 00 ]
    Krb5Context.wrap: data=[01 01 00 00 7a 6b 74 65 73 74 63 6c 69 65 6e 74 2f 6b 65 72 62 65 72 6f 73 2d 73 65 72 76 65 72 2d 30 32 2e 65 69 67 65 6e 72 6f 75 74 65 2e 63 6f 6d 40 45 49 47 45 4e 52 4f 55 54 45 2e 43 4f 4d ]
    Krb5Context.wrap: token=[05 04 00 ff 00 0c 00 00 00 00 00 00 2a d9 04 e7 01 01 00 00 7a 6b 74 65 73 74 63 6c 69 65 6e 74 2f 6b 65 72 62 65 72 6f 73 2d 73 65 72 76 65 72 2d 30 32 2e 65 69 67 65 6e 72 6f 75 74 65 2e 63 6f 6d 40 45 49 47 45 4e 52 4f 55 54 45 2e 43 4f 4d 6d 17 f8 84 22 96 99 2f 09 28 f2 5c ]

    WATCHER::

    WatchedEvent state:SaslAuthenticated type:None path:null  
    ```

    If you get a message like `WatchedEvent state:AuthFailed type:None path:null` somewhere in the output (not necessarily at the end of the output), please open an issue in this repository and provide details on the error you are getting, and provide the configuration files and paths.

### Running a ZooKeeper ensemble.



## References



Junqueira, Flavio; Reed, Benjamin. ZooKeeper: Distributed Process Coordination (Kindle Location 743). O'Reilly Media. Kindle Edition.
