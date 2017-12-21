# Installing and Setting up Kerberos on Debian Stretch (Debian 9.2)

## Introduction

Instructions for installing and configuring Kerberos v5 on an AWS EC2 Debian Stretch instance follow.

Commands that are indented are to be run not at the `bash` prompt but instead at the `kadmin` prompt that should be present at that point in the instructions.

## Steps

### Primary and alternative KDCs

Follow the steps below for primary, secondary, etc, KDCs.

1. Get a static IP address. If using AWS, this would be an Elastic IP.

2. Decide on a subdomain, e.g. `foo-01.mydomain.com`, and create a DNS entry to direct `foo-01.mydomain.com` to the above static IP address.

3. Create an EC2 instance or get a VPS. If using EC2, you might consider one of the Debian Stretch community AMIs (I chose `debian-stretch-hvm-x86_64-gp2-2017-12-17-65723`). I selected a T2 Micro with 1 GB RAM and 8 GB Magnetic Storage. For the Security Group, allow traffic to ports 22 (TCP, for SSH) and 88 (UDP, for Kerberos). Assign your static IP address to this instance.

4. SSH into your instance, and run the `update` and `upgrade` commands, and install `java`:

    ```
    sudo apt-get update && sudo apt-get upgrade
    sudo apt-get install -y default-jre
    ```

5. Change the hostname on the EC2 instance/VPS to be the FQDN `foo-01.mydomain.com`. Instructions are [available at this aws link for the Amazon Linux AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-hostname.html). For Debian Stretch, you would replace the contents of `/etc/hostname` with the following

    `foo-01.mydomain.com`

    and then reboot (`sudo shutdown -r now`)

6. Add the static ip address and subdomain to your hosts file (`/etc/hosts`):

    `w.x.y.z foo-01`

    This will allow kerberos to resolve the subdomain to the static IP address


7. To install and configure Kerberos on the EC2 instance, execute the following commands. If this is fresh install, you can omit the first lines that delete and remove:

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

8. Set up a new realm (you will have to create a master password):

   `sudo kdb5_util create -s -r MYDOMAIN.COM`


9. Edit `/etc/krb5.conf` as follows:

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

10. Make the log directory and file:

    ```
    sudo mkdir /var/log/kerberos
    sudo touch /var/log/kerberos/kadmin.log
    sudo chmod a+rw /var/log/kerberos/kadmin.log
    ```

11. Edit the `/etc/krb5kdc/kadm5.acl` access control list file, and uncomment or add this line:

    `*/admin *`

    This will give admin privileges to any principal you add with the form `x/admin@y`


12. Restart the server and the kdc:

    `sudo invoke-rc.d krb5-admin-server restart && sudo invoke-rc.d krb5-kdc restart`



13. Test that there are no tickets already:

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

While we are here, lets create principals for a `zookeeper`. Let us assume that your zookeeper server will be `zookeeper.yourdomain.com`. Execute the following (you will have to choose a password for each):

```
sudo kadmin.local
  addprinc zookeeperclient
	addprinc zookeeper/zookeeper-server-01
	addprinc zookeeper/zookeeper-server-02
	addprinc zookeeper/zookeeper-server-03
```

Export the keytabs - Zookeper servers and clients will use these:

```
  ktadd -k /etc/security/zookeeperclient.keytab zookeeperclient
  ktadd -k /etc/security/zookeeper-server-01.keytab zookeeper/zookeeper-server-01
  ktadd -k /etc/security/zookeeper-server-02.keytab zookeeper/zookeeper-server-02
  ktadd -k /etc/security/zookeeper-server-03.keytab zookeeper/zookeeper-server-03
  quit
```

The next step is to [set up and test connection to a Zookeeper ensemble](README-ZooKeeper.md).

## References

https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-hostname.html

https://debian-administration.org/article/570/MIT_Kerberos_installation_on_Debian

https://help.ubuntu.com/lts/serverguide/kerberos.html

https://serverfault.com/questions/592893/completely-uninstall-kerberos-on-ubuntu-server

https://help.ubuntu.com/community/Kerberos
