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
    # server.1=0.0.0.0:2888:3888
    # server.2=zookeeper-server-02.yourdomain.com:2888:3888
    # server.3=zookeeper-server-03.yourdomain.com:2888:3888
    authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
    requireClientAuthScheme=sasl
    jaasLoginRenew=3600000
    ```

    Note: replace `zookeeper-server-0X` with `0.0.0.0` when X is the id of the Zookeeper instance you are setting up, and all other `server.X` lines should have `zookeeper-server-0X`. See [this StackOverflow post](https://stackoverflow.com/questions/30940981/zookeeper-error-cannot-open-channel-to-x-at-election-address) for more information on why this must be the case when using EC2.

    For now, the `server.X` lines are commented out, in order to test the ZooKeeper server in StandAlone mode. Later we will uncomment these lines so that we can use ZooKeeper as an ensemble of servers.

10. Create the `dataDir` and `dataLogDir` folders, and a `jaas folder`, and create a symlink from `zoo.cfg` to `zoo.cfg-ensemble`:

    ```
    mkdir /home/zookeeper/dataDir
    mkdir /home/zookeeper/dataLogDir
    mkdir /home/zookeeper/jaas
    ln -s /home/zookeeper/zk/zookeeper-3.4.11/conf/zoo.cfg-ensemble /home/zookeeper/zk/zookeeper-3.4.11/conf/zoo.cfg
    exit
    sudo ln -s /home/zookeeper/zk/zookeeper-3.4.11 /opt/zookeeper
    sudo su - zookeeper
    cd zk/zookeeper-3.4.11
    ```

11. In the `/home/zookeeper/zk/zookeeper-3.4.11/conf/log4j.properties` file, decide where you will place log files. Junqueira & Reed (see References below) recommend writing log files to a separate drive. If you want to keep them on the same drive for now, then replace the lines:

    ```
    zookeeper.log.dir=.
    ...
    zookeeper.tracelog.dir=.
    ```

    with

    ```
    zookeeper.log.dir=/home/zookeeper/log
    ...
    zookeeper.tracelog.dir=/home/zookeeper/log
    ```

    and create the log directory:

    ```
    mkdir /home/zookeeper/log
    ```

    You will have to also figure out how to get `zkServer.sh` to stop overriding values there. [See this (seemingly dead) issue](https://issues.apache.org/jira/browse/ZOOKEEPER-2170) for more details. It seems that the `log4j.properties` file is ignored in favor of the environment variables `ZOO_LOG_DIR` and `ZOO_LOG4J_PROP`.

    If you decide to log, remember to periodically clear out the log files so that you do not run out of file storage space.

12. In the `/home/zookeeper/jaas/` folder, place the `zookeeper-server-01.keytab` file that you created when setting up Kerberos. Make the `zookeeper` user the owner of this file, and make sure this user has read and write permissions on the file. Also in this folder, create a `jaas.conf` file with the following contents:

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

13. In `/home/zookeeper/dataDir/`, create a file `myid` and place a solitary `1` in this file:

    ```
    echo "1" > /home/zookeeper/dataDir/myid
    ```

    `1` is used here because this is `zookeeper-server-01`. For `zookeeper-server-02`, one would use a `2` instead, etc.


14. In the `/etc/` folder, place the `/etc/krb5.conf` file from [the Kerberos server that you set up earlier](README-Kerberos.md), and make sure it is world readable.

15. Repeat the above steps for the other ZooKeeper servers in the ensemble (`zookeeper-server-02` and `zookeeper-server-03`)

16. To test that the server starts without errors, execute the following command from `/home/zookeeper/zk/zookeeper-3.4.11/`:

    ```
    JVMFLAGS="-Djava.security.auth.login.config=/home/zookeeper/jaas/jaas.conf -Dsun.security.krb5.debug=true" ZOO_LOG_DIR="/home/zookeeper/log" ZOO_LOG4J_PROP=TRACE,CONSOLE,ROLLINGFILE bin/zkServer.sh start-foreground
    ```

    The output should be similar to the following:

    ```
    ...
    >>> KrbAsRep cons in KrbAsReq.getReply zookeeper/zookeeper-server-01.yourdomain.com
    2017-12-26 20:46:20,231 [myid:] - INFO  [main:Login@297] - Server successfully logged in.
    2017-12-26 20:46:20,234 [myid:] - INFO  [main:NIOServerCnxnFactory@89] - binding to port 0.0.0.0/0.0.0.0:2181
    2017-12-26 20:46:20,235 [myid:] - INFO  [Thread-1:Login$1@130] - TGT refresh thread started.
    2017-12-26 20:46:20,235 [myid:] - INFO  [Thread-1:Login@305] - TGT valid starting at:        Tue Dec 26 20:46:20 UTC 2017
    2017-12-26 20:46:20,236 [myid:] - INFO  [Thread-1:Login@306] - TGT expires:                  Wed Dec 27 06:46:20 UTC 2017
    2017-12-26 20:46:20,236 [myid:] - INFO  [Thread-1:Login$1@185] - TGT refresh sleeping until: Wed Dec 27 04:58:21 UTC 2017
    ```    

### Testing the ZooKeeper server in StandAlone mode.

17. On another machine (EC2 instance, personal, whatever - as long as it is not the same machine as the server, though it can be), download ZooKeeper as before to a directory of your choosing - I'll assume you have downloaded it to your home directory:

    ```
    cd ~
    wget -c http://apache.mirror.globo.tech/zookeeper/zookeeper-3.4.11/zookeeper-3.4.11.tar.gz
    tar -zxvf zookeeper-3.4.11.tar.gz
    rm zookeeper-3.4.11.tar.gz
    cd zookeeper-3.4.11
    ```

18. Place the `/etc/security/zookeeperclient.whatever.keytab` that you created in the [Kerberos](README-Kerberos.md) section in some directory, say `~/jaas/` (create this directory). Make sure you have read permission on it.

19. Also in the `~/jaas/` section, create a file called `jaas.conf` and add the following content to it:

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
20. Place the `/etc/krb5.conf` file from the Kerberos server that you set up in the `~/jaas/` directory of the client machine, and give yourself read permissions on it.

21. From the `~/zk/zookeeper-3.4.11/` directory, run the following command:

    ```
    JVMFLAGS="-Djava.security.auth.login.config=/full/path/to/jaas/jaas.conf -Dsun.security.krb5.debug=true -Djava.security.krb5.conf=/full/path/to/jass/krb5.conf" bin/zkCli.sh -server zookeeper-server-01.yourdomain.com:2181
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

22. If you were able to SASL authenticate to the ZooKeeper server in StandAlone mode, try executing some ZooKeeper commands:

    ```
    [zk: zookeeper-server-01.yourdomain.com:2181(CONNECTED) 0] ls /
    [zookeeper]
    [zk: zookeeper-server-01.yourdomain.com:2181(CONNECTED) 1] create /mynode "some data"
    Created /mynode
    [zk: zookeeper-server-01.yourdomain.com:2181(CONNECTED) 2] ls /
    [mynode, zookeeper]
    [zk: zookeeper-server-01.yourdomain.com:2181(CONNECTED) 3] get /mynode
    some data
    cZxid = 0xb4
    ctime = Wed Dec 27 01:08:00 UTC 2017
    mZxid = 0xb4
    mtime = Wed Dec 27 01:08:00 UTC 2017
    pZxid = 0xb4
    cversion = 0
    dataVersion = 0
    aclVersion = 0
    ephemeralOwner = 0x0
    dataLength = 9
    numChildren = 0
    [zk: zookeeper-server-01.yourdomain.com:2181(CONNECTED) 4] delete /mynode
    [zk: zookeeper-server-01.yourdomain.com:2181(CONNECTED) 5] ls /
    [zookeeper]
    [zk: zookeeper-server-01.yourdomain.com:2181(CONNECTED) 6]
    ```    

    If you were able to achieve something similar to the above, then continue.

23. Repeat the above for additional ZooKeeper servers to test that you can get them each working in StandAlone mode.

### Running a ZooKeeper ensemble.

24. Stop all of the ZooKeeper servers that you created and started. On all of the servers, uncomment the `# server.X` lines. Start all of the servers.

    In the output of each of the servers, you should see one with `LEADING` and others with `FOLLOWING - LEADER ELECTION TOOK ...`. If this is the case, then you have the ensemble working correctly.

### Restricting access using ACLs.

25. At this point, anyone can connect to your ZooKeeper ensemble, and use it and abuse it. We cannot (as over version 3.4) prevent people from connecting to it, but we can prevent people from creating, reading, writing, or deleting znodes, or from modifying ACLs.

    Return to you ZooKeeper client machine, and connect without SASL:

    ```
    bin/zkCli.sh -server zookeeper-server-01.yourdomain.com:2181,zookeeper-server-02.yourdomain.com:2181,zookeeper-server-03.yourdomain.com:2181
    ```

    and play around, by creating a node, reading it, deleting it, getting the ACL of the root znode, etc:

    ```
    [...] ls /
    [zookeeper]
    [...] create /badnode "vulnerable"
    Created /badnode
    [...] ls /
    [badnode, zookeeper]
    [...] getAcl /badnode
    'world,'anyone
    : cdrwa
    [...] delete /badnode
    [...] ls /
    [zookeeper]
    [...] getAcl /
    'world,'anyone
    : cdrwa
    ```

    Let us create a new node and set permissions on it so that only our `zktestclient/whatever` user may read, write, or delete it, or create child znodes under it.

    ```
    [...] create /newnode "for zk sasl client"
    Created /newnode
    [...] ls /
    [zookeeper, newnode]
    [...] get /newnode
    for zk sasl client
    cZxid = 0x1e0000001a
    ctime = Wed Dec 27 04:48:37 UTC 2017
    mZxid = 0x1e0000001a
    mtime = Wed Dec 27 04:48:37 UTC 2017
    pZxid = 0x1e0000001a
    cversion = 0
    dataVersion = 0
    aclVersion = 0
    ephemeralOwner = 0x0
    dataLength = 18
    numChildren = 0
    [...] setAcl /newnode sasl:zktestclient/whatever@YOURDOMAIN.COM:crwd
    cZxid = 0x1e0000001a
    ctime = Wed Dec 27 04:48:37 UTC 2017
    mZxid = 0x1e0000001a
    mtime = Wed Dec 27 04:48:37 UTC 2017
    pZxid = 0x1e0000001a
    cversion = 0
    dataVersion = 0
    aclVersion = 1
    ephemeralOwner = 0x0
    dataLength = 18
    numChildren = 0
    [...] getAcl /newnode
    'sasl,'sasl:zktestclient/whatever@YOURDOMAIN.COM:crwd
    : cdrw
    [...] get /newnode
    Authentication is not valid : /newnode
    [...] create /newnode/childofnewnode "child of new node"
    Authentication is not valid : /newnode/childofnewnode
    [...] ls /
    [zookeeper, newnode]
    ```

    So we created a new znode `/newnode`, but set the ACL for it such that we cannot create child nodes under it nor can we read it - only `zktestclient/whatever` can.

26. Let us now SASL authenticate as `zktestclient/whatever`. On your client machine, execute the following command:

    ```
    JVMFLAGS="-Djava.security.auth.login.config=/full/path/to/jaas/jaas.conf -Dsun.security.krb5.debug=true -Djava.security.krb5.conf=/full/path/to/jass/krb5.conf" bin/zkCli.sh -server zookeeper-server-01.yourdomain.com:2181,zookeeper-server-02.yourdomain.com:2181,zookeeper-server-03.yourdomain.com:2181
    ```

    and reproduce the following:

    ```
    [...] ls /
    [zookeeper, newnode]
    [...] get /newnode
    for zk sasl client
    cZxid = 0x1e0000001a
    ctime = Wed Dec 27 04:48:37 UTC 2017
    mZxid = 0x1e0000001a
    mtime = Wed Dec 27 04:48:37 UTC 2017
    pZxid = 0x1e0000001a
    cversion = 0
    dataVersion = 0
    aclVersion = 1
    ephemeralOwner = 0x0
    dataLength = 18
    numChildren = 0
    [...] create /newnode/childofnewnode "child of new node"
    Created /newnode/childofnewnode
    [...] ls /
    [zookeeper, newnode]
    [...] get /newnode/childofnewnode
    child of new node
    cZxid = 0x1e0000001f
    ctime = Wed Dec 27 05:05:16 UTC 2017
    mZxid = 0x1e0000001f
    mtime = Wed Dec 27 05:05:16 UTC 2017
    pZxid = 0x1e0000001f
    cversion = 0
    dataVersion = 0
    aclVersion = 0
    ephemeralOwner = 0x0
    dataLength = 17
    numChildren = 0
    setAcl /newnode/childofnewnode sasl:zktestclient/whatever@YOURDOMAIN.COM:crwd
    ```

    We see that, now that we are authenticated as the authorized user, we can create a child znode under our znode, and read that znode and our original znode (its parent). Let us restrict access to the parent znode (`/`).

27. Set the ACL on the root znode, using any connected user. Because the  the root znode currently has the ACL `world:anyone:cdrwa`, anyone can set its ACL.

    ```
    setAcl /newnode/childofnewnode sasl:zktestclient/whatever@YOURDOMAIN.COM:crwd
    ```    

    The root znode is now restricted to only `zktestclient/whatever` and the super user. Remember to set the ACLs as above for all nodes that you create.

28. Try connecting to the other ZooKeper servers to see whether they hold the same state:

    ```
    JVMFLAGS="-Djava.security.auth.login.config=/full/path/to/jaas/jaas.conf -Dsun.security.krb5.debug=true -Djava.security.krb5.conf=/full/path/to/jass/krb5.conf" bin/zkCli.sh -server zookeeper-server-03.yourdomain.com:2181
    ```

    Get the ACL for `/newnode/childofnewnode`, etc. Do this for three nodes. The results should be the same.

29. For all of your ZooKeper servers, make ZooKeeper as service. There is a nice `init.d` script [here](https://gist.github.com/bejean/b9ff72c6d2143e16e35d) that you can adapt. More info from [debian-administration.org is here](https://debian-administration.org/article/28/Making_scripts_run_at_boot_time_with_Debian). I have copied the aforementioned script [here - zookeeper](zookeeper), and adapted it slightly. Execute the following to set up ZooKeeper as a service:

    In /home/zookeeper/.profile, add the following:

    ```
    export ZOO_LOG_DIR="/home/zookeeper/log"
    export ZOO_JAAS_CONF="/home/zookeeper/jaas/jaas.conf"
    export JVMFLAGS="-Djava.security.auth.login.config=$ZOO_JAAS_CONF"
    export ZOO_LOG4J_PROP=TRACE,ROLLINGFILE
    ```

    Create an init.d script for ZooKeper:

    ```
    sudo vi /etc/init.d/zookeeper
    ```

    Paste the [zookeeper](zookeeper) script in the editor and save it.

    ```
    sudo chmod 755 /etc/init.d/zookeeper
    sudo update-rc.d zookeeper defaults
    sudo service zookeeper start
    ```

    Now ZooKeeper should start automatically after rebooting.

### ZooKeper super user (optional)

30. ACLs do not apply to super users. Instructions for activating and authenticating as a super user follow. On one of the ZooKeeper servers we will activate the super user login and set the super user password. Choose a ZooKeeper server machine, and execute the following from the `~/zk/zookeeper-3.4.11/` directory in a different bash (terminal) session than the one that is running the server:

    ```
    export ZK_CLASSPATH=/home/zookeeper/zk/zookeeper-3.4.11/bin/../build/classes:/home/zookeeper/zk/zookeeper-3.4.11/bin/../build/lib/*.jar:/home/zookeeper/zk/zookeeper-3.4.11/bin/../lib/slf4j-log4j12-1.6.1.jar:/home/zookeeper/zk/zookeeper-3.4.11/bin/../lib/slf4j-api-1.6.1.jar:/home/zookeeper/zk/zookeeper-3.4.11/bin/../lib/netty-3.10.5.Final.jar:/home/zookeeper/zk/zookeeper-3.4.11/bin/../lib/log4j-1.2.16.jar:/home/zookeeper/zk/zookeeper-3.4.11/bin/../lib/jline-0.9.94.jar:/home/zookeeper/zk/zookeeper-3.4.11/bin/../lib/audience-annotations-0.5.0.jar:/home/zookeeper/zk/zookeeper-3.4.11/bin/../zookeeper-3.4.11.jar:/home/zookeeper/zk/zookeeper-3.4.11/bin/../src/java/lib/*.jar:/home/zookeeper/zk/zookeeper-3.4.11/bin/../conf:
    java -cp $ZK_CLASSPATH org.apache.zookeeper.server.auth.DigestAuthenticationProvider super:some-secret-password
    ```

    (You can get the class path by running `zkEnv.sh` after uncommenting the last line in this file, which echo's the class path to the terminal).

    The output should be similar to:

    ```
    super:some-secret-password->super:Bl5S86TbxiWTRBCdXR1pfGuau48=
    ```

    Stop the ZooKeeper server on this machine, and start it again using the following command:

    ```
    JVMFLAGS="-Djava.security.auth.login.config=/ho/zookeeper/jaas/jaas.conf -Dsun.security.krb5.debug=true -Dzookeeper.DigestAuthenticationProvider.superDigest=super:Bl5S86TbxiWTRBCdXR1pfGuau48=" ZOO_LOG_DIR="/home/zookeeper/log" ZOO_LOG4J_PROP=TRACE,ROLLINGFILE,CONSOLE  bin/zkServer.sh start-foreground
    ```

31. In a separate bash session on the same machine that you are running this ZooKeper server instance with super user enabled (you can use the same session that you used above to generate the super user password), connect to this ZooKeeper server using a non-SASL authenticated client:

    ```
    cd ~/zk/zookeeper-3.4.11
    bin/zkCli.sh -server localhost:2181
    ```

    Note that this has to be done on the same machine as the ZooKeper server, since traffic between ZooKeper servers and clients is unencrypted (as of ZooKeper 3.4), so you want to avoid sending the super user password over an unsecured network.

32. Authenticate as the super user:

    ```
    authinfo digest super:some-secret-password
    ```

    You can now set ACLs, obtain information on any znode by running `get` on that znode, etc.

## References

Junqueira, Flavio; Reed, Benjamin. ZooKeeper: Distributed Process Coordination (Kindle Location 743). O'Reilly Media. Kindle Edition.

https://stackoverflow.com/questions/30940981/zookeeper-error-cannot-open-channel-to-x-at-election-address

https://gist.github.com/bejean/b9ff72c6d2143e16e35d

https://debian-administration.org/article/28/Making_scripts_run_at_boot_time_with_Debian

https://issues.apache.org/jira/browse/ZOOKEEPER-2170
