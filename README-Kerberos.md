# Installing and Setting up Kerberos on Debian Stretch (Debian 9.2)

## Introduction

Instructions for installing and configuring Kerberos v5 on an AWS EC2 Debian Stretch instance follow.

Commands that are indented are to be run not at the `bash` prompt but instead at the `kadmin` prompt that should be present at that point in the instructions.

## Steps

### Primary and alternative KDCs

Follow the steps below for primary, secondary, etc, KDCs.

1. Get a static IP address. If using AWS, this would be an Elastic IP. If using Google Cloud, you would reserve a static IP address and then link it to your VM instance.

2. Decide on a subdomain, e.g. `foo-01.mydomain.com`, and create a DNS entry to direct `foo-01.mydomain.com` to the above static IP address.

3. Create an EC2 instance, Google Cloud VM instance,  or get a VPS. If using EC2, you might consider one of the Debian Stretch community AMIs (I chose `debian-stretch-hvm-x86_64-gp2-2017-12-17-65723`). I selected a T2 Micro with 1 GB RAM and 8 GB Magnetic Storage. For the Security Group, allow traffic to ports 22 (TCP, for SSH) and 88 (UDP, for Kerberos). Assign your static IP address to this instance.

    If using Google Compute Cloud, currently Debian 9 Stretch is the default VM image.

4. This step is relevant only for ZooKeeper servers, but might be useful for Kerberos and Kafka servers as well: make sure that a reverse DNS look up on your static IP address resolves to `foo-01.mydomain.com`. If you're using AWS, see [here](https://aws.amazon.com/blogs/aws/reverse-dns-for-ec2s-elastic-ip-addresses/) for how to request AWS to set up the mapping for a reverse DNS lookup. If a reverse DNS lookup does not resolve to `foo-01.mydomain.com`, then connecting to your ZooKeeper Server from the ZooKeeper client will not work.

    If using Google Cloud, the process is easier - you can [create a PTR record](https://cloud.google.com/compute/docs/instances/create-ptr-record) and then verify it by creating a TXT record as per the instructions you will get at the page for creating a PTR record.  

5. Make sure that port 22 is open. SSH into your instance, and run the `update` and `upgrade` commands, and install `java`:

    ```
    sudo apt-get update && sudo apt-get -y upgrade
    sudo apt-get install -y default-jre
    ```

6. Change the hostname on the EC2 instance/VPS to be the FQDN `foo-01.mydomain.com`. Instructions are [available at this aws link for the Amazon Linux AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-hostname.html). For Debian Stretch, you would replace the contents of `/etc/hostname` with the following

    `foo-01.mydomain.com`

    and then reboot (`sudo shutdown -r now`)

    On Google Cloud, this step is not necessary.

7. (a) For Kerberos servers (this is not necessary for ZooKeeper servers and Kafka servers, though you can do this for those too), add the static IP address and subdomain to your hosts file (`/etc/hosts`):

    `w.x.y.z foo-01`

    This will allow kerberos to resolve the subdomain to the static IP address.

    (b) For Kafka nodes, and optionally for all other nodes, also add a line with the loop-back address and the fully qualified domain name:

    `127.0.0.1 foo-01.mydomain.com`


8. To install and configure Kerberos on the EC2 instance, execute the following commands. If this is fresh install, you can omit the first lines that delete and remove:

    ```
    sudo rm -rf /etc/krb5conf
    sudo rm -rf /var/lib/krb5kdc
    sudo rm -rf /etc/krb5.conf
    sudo apt purge -y krb5-kdc krb5-admin-server krb5-config krb5-locales krb5-user krb5.conf
    sudo apt-get install -y krb5-{kdc,admin-server}
    ```

    This last command will prompt you for information. Accept the REALM it chooses (MYDOMAIN.COM). Enter the hostnames of all of the servers:

    `foo-01.mydomain.com foo-02.mydomain.com foo-03.mydomain.com`

    If you mess things up, re-execute the above commands. Likely, you will get an error after executing the above along the lines of `Cannot open DB2 database '/var/li…`. That's OK - continue with the steps below.

9. Set up a new realm (you will have to create a master password):

   `sudo kdb5_util create -s -r MYDOMAIN.COM`


10. Edit `/etc/krb5.conf` as follows:

    Under `[libdefaults]` add: `rdns = false`

    Tack the following on to the end of the file (starting under [domain_realm]

    ```      
    .mydomain.com = MYDOMAIN.COM
    mydomain.com = MYDOMAIN.COM

    [logging]
       kdc = FILE:/var/log/kerberos/krb5kdc.log
       admin_server = FILE:/var/log/kerberos/kadmin.log
       default = FILE:/var/log/kerberos/krb5lib.log
    ```

11. Create the log directory and file:

    ```
    sudo mkdir /var/log/kerberos
    sudo touch /var/log/kerberos/kadmin.log
    sudo chmod a+rw /var/log/kerberos/kadmin.log
    ```

12. Edit the `/etc/krb5kdc/kadm5.acl` access control list file, and uncomment or add this line:

    `*/admin *`

    This will give admin privileges to any principal you add with the form `x/admin@y`

13. Restart the server and the kdc:

    `sudo invoke-rc.d krb5-admin-server restart && sudo invoke-rc.d krb5-kdc restart`


14. Test that there are no tickets already:

    `klist # should output: klist: No credentials cache found (filename: /tmp/krb5cc_1000)`

### Primary KDC

If this will be the Primary KDC, then create an admin principal (it will prompt you for a user password):

```
sudo kadmin.local
	addprinc yourname/admin
	quit
```

Create a ticket:

```
KRB5_TRACE=/dev/stdout kinit yourname/admin
```

Test that the ticket was created:

```
klist  # should output some issued date, expire date, etc.   
```

While we are here, lets create principals for our ZooKeeper ensemble and our Kafka cluster. Let us assume that your zookeeper server will be `zookeeper-server-0X.yourdomain.com`. Execute the following (you will have to choose a password for each):

```
sudo kadmin.local
  addprinc zookeeperclient/whatever
	addprinc zookeeper/zookeeper-server-01.yourdomain.com
	addprinc zookeeper/zookeeper-server-02.yourdomain.com
	addprinc zookeeper/zookeeper-server-03.yourdomain.com
  addprinc kafka/server-01.yourdomain.com
  addprinc kafka/server-02.yourdomain.com
  addprinc consumer1/whatever
  addprinc producer1/whatever
```

Export the keytabs - Zookeeper servers and clients will use these (you can name the `keytab` files whatever you want, you don't have to follow the convention I have used below):

  ```
  ktadd -k /etc/security/zookeeperclient.whatever.keytab zookeeperclient/whatever
  ktadd -k /etc/security/zookeeper.zookeeper-server-01.yourdomain.com.keytab zookeeper/zookeeper-server-01.yourdomain.com
  ktadd -k /etc/security/zookeeper.zookeeper-server-02.yourdomain.com.keytab zookeeper/zookeeper-server-02.yourdomain.com
  ktadd -k /etc/security/zookeeper.zookeeper-server-03.yourdomain.com.keytab zookeeper/zookeeper-server-03.yourdomain.com
  ktadd -k /etc/security/kafka.server-01.yourdomain.com.keytab kafka/server-01.yourdomain.com
  ktadd -k /etc/security/kafka.server-02.yourdomain.com.keytab kafka/server-02.yourdomain.com
  ktadd -k /etc/security/consumer1.whatever.keytab consumer1/whatever
  ktadd -k /etc/security/producer1.whatever.keytab producer1/whatever
  quit
  ```

The next step is to [set up and test connection a to a Zookeeper ensemble](README-Zookeeper.md).

## References

https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-hostname.html

https://debian-administration.org/article/570/MIT_Kerberos_installation_on_Debian

https://help.ubuntu.com/lts/serverguide/kerberos.html

https://serverfault.com/questions/592893/completely-uninstall-kerberos-on-ubuntu-server

https://help.ubuntu.com/community/Kerberos

https://aws.amazon.com/blogs/aws/reverse-dns-for-ec2s-elastic-ip-addresses
