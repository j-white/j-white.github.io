---
layout: post
title:  "Distributed Monitoring with OpenNMS Minion"
date:   2016-12-18 19:22:17
categories: [opennms, minion]
---

In _OpenNMS_ Horizon 19.0.0 we've added support for distributed monitoring, making it possible to monitor networks and devices which can't be directly reached from the _OpenNMS_ instance.
This is achieved by leveraging a new service we call _Minion_.


_Minions_ take the role of communicating with the managed devices, while _OpenNMS_ performs the coordination and task delegation.
The _Minions_ can operate behind firewall and/or NAT, as long as they can communicate with _OpenNMS_ via REST (backed by _Jetty_) and JMS (provided by an embedded _ActiveMQ_ broker).


In this guide we'll walk you through setting up instances of both _OpenNMS_ and _Minion_ on [DigitalOcean] and walk you through securing the communication channels using SSL.

### Bootstrapping OpenNMS

For this setup, I created a new [Droplet] with 2GB RAM and 40 GB of disk space, based on CentOS 7.2 x64 with a hostname of `nms01.jessewhite.ca`.

  > The hostname will come in handy when we setup SSL later

Once your instance is ready, fuel up the entropy pools by installing _haveged_:

    yum -y install epel-release
    yum -y install haveged
    systemctl enable haveged
    systemctl start haveged

  > OpenNMS uses a lot of entropy on startup for initializing the SNMPv3 contexts (even when SNMPv3 isn't used).
  > Startup will take considerably longer if it needs to block while waiting for entropy.

Next, we want to install the _OpenNMS_ YUM repository:

    rpm -Uvh https://yum.opennms.org/repofiles/opennms-repo-branches-release-19.0.0-rhel7.noarch.rpm
    rpm --import https://yum.opennms.org/OPENNMS-GPG-KEY

  > As I write this, 19.0.0 isn't currently released, so we use the latest 19.0.0 release snapshots.
  > Once 19.0.0 is released, you should change the URL to the RPM above to `https://yum.opennms.org/repofiles/opennms-repo-stable-rhel7.noarch.rpm`.

Next, follow the instructions in the installation guide for [Installing OpenNMS and PostgreSQL].
For reference, I ran the following:

     yum -y install opennms
     postgresql-setup initdb
     systemctl enable postgresql
     systemctl start postgresql
     cd /tmp
     sudo -u postgres bash -c "psql -c \"CREATE USER opennms WITH PASSWORD 'op3nnms';\""
     sudo -u postgres bash -c "createdb -O opennms opennms"
     sudo -u postgres bash -c "psql -c \"ALTER USER postgres WITH PASSWORD 'op4nnms';\""
     sed -i 's/ident/md5/' /var/lib/pgsql/data/pg_hba.conf
     systemctl reload postgresql
     sed -i 's/password="opennms"/password="op3nnms"/' /opt/opennms/etc/opennms-datasources.xml
     sed -i 's/password=""/password="op4nnms"/' /opt/opennms/etc/opennms-datasources.xml
     /opt/opennms/bin/runjava -s
     /opt/opennms/bin/install -dis
     systemctl enable opennms

There's currently an issue with some of the _Karaf_ features from the 19.0.0 release snapshots that can cause startup and shutdown to hang.
This may not be an issue by the time you read this, but this can currently be remedied by removing the problematic features from startup:

    sed -i '/opennms-.*-handler-default/d' /opt/opennms/etc/org.apache.karaf.features.cfg

With everything in place, we can now start OpenNMS for the first time

     systemctl start opennms

After starting OpenNMS, you may need to wait a minute or two before being able to access the Web UI. While waiting, I typically run:

    tail -f /opt/opennms/logs/manager.log

And wait for a message of the form:

    2016-12-18 15:59:13,509 DEBUG [Main] o.o.n.v.Starter: Startup complete

Once startup is complete, you should be able to login to your instance on port 8980 using the default `admin/admin` credentials.
In my case, I can login using `http://nms01.jessewhite.ca:8980/`.

### Enabling SSL

#### Obtaining an SSL certificate using _Let's Encrypt_

If you don't already have an SSL certificate, you can obtain one freely via _Let's Encrypt_.
To do this on our _Droplet_, we leverage [certbot]:

    yum -y install certbot

  > If you installed haveged above, the required EPEL repository should already be installed and enabled.

Generate a new SSL certificate:

    certbot certonly --standalone -d nms01.jessewhite.ca

Copy the SSL certificate and private key to a Java KeyStore:

```sh
echo '#!/bin/sh
HOSTNAME="nms01.jessewhite.ca"

TEMP_P12=$(mktemp -p /tmp ssl.p12.XXXXXXX)
TEMP_KEYSTORE=$(mktemp -p /tmp ssl.keystore.XXXXXXX)
TARGET_KEYSTORE="/opt/opennms/etc/opennms.letsencrypt.jks"

openssl pkcs12 -export -in /etc/letsencrypt/live/$HOSTNAME/fullchain.pem -inkey /etc/letsencrypt/live/$HOSTNAME/privkey.pem -out $TEMP_P12 -name opennms -password pass:changeit
# keytool complains if the file already exists
rm -f $TEMP_KEYSTORE
keytool -importkeystore -deststorepass changeit -destkeypass changeit -destkeystore $TEMP_KEYSTORE -srckeystore $TEMP_P12 -srcstoretype PKCS12 -srcstorepass changeit -alias opennms

cp $TEMP_KEYSTORE $TARGET_KEYSTORE
chmod 440 $TARGET_KEYSTORE
rm -f $TEMP_P12
rm -f $TEMP_KEYSTORE
' > /opt/opennms/bin/pem-to-jks.sh
chmod +x /opt/opennms/bin/pem-to-jks.sh
/opt/opennms/bin/pem-to-jks.sh
```

We should now have a valid SSL certificate stored in `/opt/opennms/etc/opennms.letsencrypt.jks`.

#### Protecting Jetty using SSL

Next, let's enable the HTTPS connector in _Jetty_ by editing `jetty.xml`:

    cp /opt/opennms/etc/examples/jetty.xml /opt/opennms/etc/jetty.xml
    vi /opt/opennms/etc/jetty.xml

Here we only need to uncomment the block after `<!-- Add HTTPS support -->`.
Here's a `diff` after my changes:

```sh
diff -rupN etc/examples/jetty.xml etc/jetty.xml
--- etc/examples/jetty.xml      2016-12-16 23:20:28.000000000 +0000
+++ etc/jetty.xml       2016-12-18 16:03:52.161000000 +0000
@@ -40,7 +40,6 @@
   -->

   <!-- Add HTTPS support -->
-  <!--
   <New id="sslHttpConfig" class="org.eclipse.jetty.server.HttpConfiguration">
     <Arg><Ref refid="httpConfig"/></Arg>
     <Call name="addCustomizer">
@@ -112,7 +111,6 @@
       </New>
     </Arg>
   </Call>
-  -->

   <!-- Note #1 To set X-Frame-Options uncomment here and uncomment #2 below  -->
   <!-- Settings for X-Frame-Options to avoid clickjacking                    -->
```

The remaining flags can be set using system properties:

```sh
echo 'org.opennms.netmgt.jetty.host = 127.0.0.1
org.opennms.netmgt.jetty.https-port = 8443
org.opennms.netmgt.jetty.https-keystore = /opt/opennms/etc/opennms.letsencrypt.jks
org.opennms.netmgt.jetty.https-keystorepassword = changeit
org.opennms.netmgt.jetty.https-keypassword = changeit' > /opt/opennms/etc/opennms.properties.d/https.properties
```

After restarting OpenNMS, we should now be able to access the Web UI via HTTPS.
In my case, I can login using `https://nms01.jessewhite.ca:8443/`.

#### Protecting ActiveMQ using SSL

To enable SSL in _ActiveMQ_ we need to define an SSL context and enable the SSL connector.
We can do this by editing:

    vi /opt/opennms/etc/opennms-activemq.xml

Here's a `diff` after my changes:

```sh
diff -rupN share/etc-pristine/opennms-activemq.xml etc/opennms-activemq.xml
--- share/etc-pristine/opennms-activemq.xml     2016-12-16 23:29:45.000000000 +0000
+++ etc/opennms-activemq.xml    2016-12-18 16:23:30.948000000 +0000
@@ -67,6 +67,10 @@
           </authorizationPlugin>
         </plugins>

+        <sslContext>
+         <sslContext keyStore="/opt/opennms/etc/opennms.letsencrypt.jks" keyStorePassword="changeit"/>
+        </sslContext>
+
         <!--
             For better performances use VM cursor and small memory limit.
             For more information, see:
@@ -167,7 +171,7 @@
               WARNING: Access to port 61616 should be firewalled to prevent unauthorized injection
               of data into OpenNMS when this port is open.
             -->
-            <!-- <transportConnector name="openwire" uri="tcp://0.0.0.0:61616?useJmx=false&amp;maximumConnections=1000&amp;wireformat.maxFrameSize=104857600"/> -->
+            <transportConnector name="openwire" uri="ssl://0.0.0.0:61616?useJmx=false&amp;maximumConnections=1000&amp;wireformat.maxFrameSize=104857600"/>

             <!-- Uncomment this line to allow localhost TCP connections (for testing purposes) -->
             <!-- <transportConnector name="openwire" uri="tcp://127.0.0.1:61616?useJmx=false&amp;maximumConnections=1000&amp;wireformat.maxFrameSize=104857600"/> -->
```

After restarting _OpenNMS_, we should be able to connect to `nms01.jessewhite.ca` on port 61616 using SSL:

    openssl s_client -connect nms01.jessewhite.ca:61616

### Managing users

Now that all of our communications are encrypted, we can change default credentials and create a new user for our Minion.
We're going to go through these quickly.
See [User Management] if you need more detailed instructions.

#### admin user

Login as the `admin` user, and change the default password using `Change Password` option on the user menu.

#### rtc user

Login as the `admin` user and change the default password for the `rtc` user.

Update the configuration with the changed password:

```sh
echo 'opennms.rtc-client.http-post.password = !rtc' > /opt/opennms/etc/opennms.properties.d/rtc.properties
chmod 660 /opt/opennms/etc/opennms.properties.d/rtc.properties
```

#### minion user

Login as the `admin` user and create a new user called `minion`.
Add the `MINION` role, to the newly created `minion` user.

  > The `MINION` role includes the minimal permissions required for operating a _Minion_.
  > These include read-only access to the necessary REST endpoints, and access the the necessary queues in _ActiveMQ_.

### Bootstrapping Minion

Now that our _OpenNMS_ server is up and secure, we can focus on setting up our _Minion_.

For this setup, I created a new [Droplet] with 1GB RAM and 30 GB of disk space, based on CentOS 7.2 x64 with a hostname of `minion01.jessewhite.ca`.

Follow the same instructions above for setting up `haveged`, and installing the _OpenNMS_ YUM repository.

Before installing _Minion_, we going to setup a simple network that can be reached from the _Minion_ instance, but can't be reached from the _OpenNMS_ instance.

    ip tuntap add tap0 mode tap
    ip addr add 172.16.1.1/24 dev tap0
    ip addr add 172.16.1.2/24 dev tap0
    ip addr add 172.16.1.3/24 dev tap0

Now we can quickly verify that it's reachable from the _Minion_, but not from _OpenNMS_:

    ping -c 1 172.16.1.1

Next, let's install and start _Minion_:

    yum -y install opennms-minion
    systemctl enable minion
    systemctl start minion

Once _Minion_ has started, you can login via the _Karaf_ shell:

    ssh -p 8201 admin@127.0.0.1

  > Default credentials are admin/admin

Let's point _Minion_ to our _OpenNMS_ instance now:

    config:edit org.opennms.minion.controller
    config:property-set http-url https://nms01.jessewhite.ca:8443/opennms
    config:property-set broker-url failover://ssl://nms01.jessewhite.ca:61616
    config:property-set location SITE01
    config:update

And provide the _Minion_ with the required credentials:

    scv:set opennms.http minion pa$$w0rd
    scv:set opennms.broker minion pa$$w0rd

  > The password here should be the one we set when we create the `minion` user above.

Restart the _Minion_:

    systemctl restart minion

Once _Minion_ is back online, log back into the _Karaf_ shell and use the `minion:ping` command to verify communication with the _OpenNMS_ instance:

    minion:ping

If everything works properly, you should see output similar to:

```
admin@minion> minion:ping
Connecting to ReST...
OK
Connecting to Broker...
OK
admin@minion>
```

If you see output similar to:

```
admin@minion> minion:ping
Connecting to ReST...
Error executing command: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
admin@minion>
```

then your `JDK` does not trust the SSL certificates installed on _OpenNMS_.
If your using certificates from _Let's Encrypt_, this can be fixed by upgrade your `JDK` to `8u101` or greater.
Alternatively, you can import the certificate chain into your `JDK`'s trust store:

    /usr/java/latest/bin/keytool -trustcacerts -keystore /usr/java/latest/jre/lib/security/cacerts -storepass changeit -noprompt -importcert -file fullchain.pem

### Securing minion

Before we forget, let's change the default password for the 'admin' user used to access the _Karaf_ shell on the _Minion_.

From the _Karaf_ shell, enable password encryption using:

    config:edit org.apache.karaf.jaas
    config:property-set encryption.enabled true
    config:property-set encryption.algorithm SHA-512
    config:update

Now change the password for the 'admin' user:

    jaas:realm-manage --index 1 --realm karaf
    jaas:user-add admin adm1n
    jaas:update

Logout and try login with your new password.

### Configuring OpenNMS

So far we an setup an instance of _OpenNMS_, an instance of _Minion_, secured the communication channels and changed all of the default passwords.
We're ready to start monitoring!

#### Personalizing your Minion

If we log into the _OpenNMS_ UI, and navigate to `Info -> Nodes`, we should be brought to the node page for our _Minion_.
_Minions_ are automatically provisioned in _OpenNMS_ under the `Minion` provisioning requisition when they first connect.
The _Minion's_ node label should be a `UUID`, but we can change that to something more descriptive by changing the node's label in the provisioning requisition.

![node]({{ site.url }}/assets/2016-12-18-opennms-minion-node.png)

To change the label, navigate to `Admin -> Configure OpenNMS -> Manage Provisioning Requisitions` and then:

1. Edit the `Minions` requisition.
1. Edit the first node.
1. Modify the node label.
1. Save your changes by clicking 'Save'.
1. Return the requisition view by clicking 'Return'.
1. Synchronize the requisition by clicking 'Synchronize'.

#### Discovering your networks

Personally, I like to keep all of my nodes in requisitions, so let's create a new requisition called 'SITE01' to store the nodes.
Navigate to `Admin -> Configure OpenNMS -> Manage Provisioning Requisitions` and then:

1. Create a new requisition by clicking 'Add Requisition'.
1. Enter the requisition name and click 'OK'.
1. Synchronize the empty requisition by clicking 'Synchronize'.

Now, let's setup our discovery range, and configure the discovery to add the discovered nodes to our newly created requisition.
Navigate `Admin -> Configure OpenNMS -> Configure Discovery` and then:

1. Add a new range, scanning from `172.16.1.1` to `172.16.1.10`.
1. Use the `SITE01` foreign source, matching the name of the requisition we just created.
1. Use the `SITE01` location, matching the name of the location for which the _Minion_ is configured.
1. Start scanning by clicking `Save and Restart Discovery`.

![discovery configuration]({{ site.url }}/assets/2016-12-18-opennms-discovery-config.png)

After a couple minutes, we can navigate back to the `Info -> Nodes`, and we should now see three new nodes, one for each address we added to the `tap0` interface.
We can now simulate taking a node off-line by deleting the IP address.

    ip addr del 172.16.1.2/24 dev tap0

Now wait for up to 6 minutes (we poll every 5 minutes by default), and the node with IP address `172.16.1.2` should show up as off-line.
Restore the IP address, and the services on `172.16.1.2` should be go back on-line.

    ip addr add 172.16.1.2/24 dev tap0

### Conclusion

You're now ready to monitor your remote networks while harnessing the power of a centralized _OpenNMS_ platform.
Consult the [Administrators Guide] for additional help in configuring your _OpenNMS_ instance.
If you need any more help, you can reach out to the OpenNMS community on the mailing lists, on IRC, or in [chat].

[DigitalOcean]: https://www.digitalocean.com/
[Droplet]: https://www.digitalocean.com/products/compute/
[Installing OpenNMS and PostgreSQL]: http://docs.opennms.org/opennms/branches/release-19.0.0/guide-install/guide-install.html#gi-install-opennms-rhel
[certbot]: https://certbot.eff.org/
[User Management]: http://docs.opennms.org/opennms/branches/release-19.0.0/guide-admin/guide-admin.html#ga-role-user-management-users
[Administrators Guide]: http://docs.opennms.org/opennms/branches/release-19.0.0/guide-admin/#ga-service-assurance
[chat]: https://chat.opennms.com
