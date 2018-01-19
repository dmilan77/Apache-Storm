## Install jce (Java Cryptography Extension (JCE) Unlimited Strength )

[Download Link](http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html)  

```bash
#Extract to /usr/java/jdk1.8.0_151/jre/lib/security
[root@kerberos security]# ls -l *.jar
-rw-r--r--. 1 root root 3035 Jan 19 04:15 local_policy.jar
-rw-r--r--. 1 root root 3023 Jan 19 04:15 US_export_policy.jar
```

## Linux useradd

```bash
useradd -m zookeeper
useradd -m storm
```

## KDC Configurartions

```bash
# KDC default REALM EXAMPLE.COM

# Zookeeper
kadmin.local -q 'addprinc zookeeper/kerberos.example.com@EXAMPLE.COM'
kadmin.local -q "ktadd -k /etc/keytab/zookeeper.keytab zookeeper/kerberos.example.com@EXAMPLE.COM"
# Nimbus and DRPC
sudo kadmin.local -q 'addprinc storm/kerberos.example.com@EXAMPLE.COM'
sudo kadmin.local -q "ktadd -k /etc/keytab/storm.keytab storm/kerberos.example.com@EXAMPLE.COM"
# All UI logviewer and Supervisors
sudo kadmin.local -q 'addprinc storm@EXAMPLE.COM'
sudo kadmin.local -q "ktadd -k /etc/keytab/storm.keytab storm@EXAMPLE.COM"

# All UI logviewer and Supervisors
sudo kadmin.local -q 'addprinc HTTP@EXAMPLE.COM'
sudo kadmin.local -q "ktadd -k /etc/keytab/HTTP.keytab HTTP@EXAMPLE.COM"
```

## ​Securing ZooKeeper with Kerberos

`vi /opt/myapps/zookeeper-3.4.6/conf/jaas.conf`
```bash
Server {
  com.sun.security.auth.module.Krb5LoginModule required
  useKeyTab=true
  keyTab="/opt/myapps/zookeeper-3.4.6/conf/zookeeper.keytab"
  storeKey=true
  useTicketCache=false
  principal="zookeeper/kerberos.example.com@EXAMPLE.COM";
};
```

`vi /opt/myapps/zookeeper-3.4.6/conf/java.env`
`export JVMFLAGS="-Djava.security.auth.login.config=/opt/myapps/zookeeper-3.4.6/conf/jaas.conf"`


##  Generate Self Signed SSL Cert


```bash
DIR=/opt/myapps/apache-storm-1.1.1/certs
keytool -keystore $DIR/server.keystore.jks -alias localhost -validity 365 -keyalg RSA -genkey
openssl req -new -x509 -keyout $DIR/ca-key -out $DIR/ca-cert -days 365
keytool -keystore $DIR/server.truststore.jks -alias CARoot -import -file $DIR/ca-cert
keytool -keystore $DIR/client.truststore.jks -alias CARoot -import -file $DIR/ca-cert
keytool -keystore $DIR/server.keystore.jks -alias localhost -certreq -file $DIR/cert-file
openssl x509 -req -CA $DIR/ca-cert -CAkey $DIR/ca-key -in $DIR/cert-file -out $DIR/cert-signed -days 365 -CAcreateserial -passin pass:mypassword123
keytool -keystore $DIR/server.keystore.jks -alias CARoot -import -file $DIR/ca-cert
keytool -keystore $DIR/server.keystore.jks -alias localhost -import -file $DIR/cert-signed
```

## Storm Configurartions


`vi /opt/myapps/apache-storm-1.1.1/conf/storm.yaml`

```bash
ui.filter: "org.apache.hadoop.security.authentication.server.AuthenticationFilter"
ui.filter.params:
   "type": "kerberos"
   "kerberos.principal": "HTTP@EXAMPLE.COM"
   "kerberos.keytab": "/opt/myapps/apache-storm-1.1.1/keytab/HTTP.keytab"
   "kerberos.name.rules": "RULE:[2:$1@$0]([jt]t@.*EXAMPLE.COM)s/.*/$MAPRED_USER/ RULE:[2:$1@$0]([nd]n@.*EXAMPLE.COM)s/.*/$HDFS_USER/DEFAULT"
storm.thrift.transport: "org.apache.storm.security.auth.kerberos.KerberosSaslTransportPlugin"
java.security.auth.login.config: "/opt/myapps/apache-storm-1.0.5/conf/jaas.conf"
#nimbus.childopts: "-Xmx1024m -Djava.security.auth.login.config=/opt/myapps/apache-storm-1.1.1/conf/jaas.conf"
nimbus.childopts: "-Xmx1024m -Djava.security.auth.login.config=/opt/myapps/apache-storm-1.1.1/conf/jaas.conf -Djava.security.krb5.conf=/opt/myapps/apache-storm-1.1.1/keytab/krb5.conf"
ui.childopts: "-Xmx768m -Djava.security.auth.login.config=/opt/myapps/apache-storm-1.1.1/conf/jaas.conf"
supervisor.childopts: "-Xmx256m -Djava.security.auth.login.config=/opt/myapps/apache-storm-1.1.1/conf/jaas.conf"
storm.principal.tolocal: "org.apache.storm.security.auth.KerberosPrincipalToLocal"
storm.zookeeper.superACL: "sasl:${nimbus-user}"
nimbus.authorizer: "org.apache.storm.security.auth.authorizer.SimpleACLAuthorizer"
drpc.https.port: 9443
drpc.https.keystore.password:  "mypassword123"
drpc.https.keystore.type: "JKS"
drpc.https.keystore.path: /opt/myapps/apache-storm-1.1.1/certs/server.keystore.jks
drpc.https.keystore.password: "mypassword123"
drpc.https.key.password: "mypassword123"
drpc.https.truststore.path: /opt/myapps/apache-storm-1.1.1/certs/client.truststore.jks
drpc.https.truststore.password: "mypassword123"
drpc.https.truststore.type:  "JKS"
ui.https.port: 8443
ui.https.keystore.type: "JKS"
ui.https.keystore.path: /opt/myapps/apache-storm-1.1.1/certs/server.keystore.jks
ui.https.keystore.password: "mypassword123"
ui.https.key.password: "mypassword123"
ui.https.truststore.path: /opt/myapps/apache-storm-1.1.1/certs/client.truststore.jks
ui.https.truststore.password : "mypassword123"
ui.https.truststore.type: "JKS"
ui.https.want.client.auth: false
ui.https.need.client.auth: false
```

## Storm jaas.conf

``vi /opt/myapps/apache-storm-1.1.1/conf/jaas.conf`

```yaml
StormServer {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="/opt/myapps/apache-storm-1.1.1/keytab/storm.keytab"
   storeKey=true
   useTicketCache=false
   principal="storm/kerberos.example.com@EXAMPLE.COM";
};
StormClient {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="/opt/myapps/apache-storm-1.1.1/keytab/storm.keytab"
   storeKey=true
   useTicketCache=false
   serviceName="storm"
   principal="storm@EXAMPLE.COM";
};
Client {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="/opt/myapps/apache-storm-1.1.1/keytab/storm.keytab"
   storeKey=true
   useTicketCache=false
   serviceName="zookeeper"
   principal="storm@EXAMPLE.COM";
};
Server {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="/opt/myapps/apache-storm-1.1.1/keytab/zookeeper.keytab"
   storeKey=true
   useTicketCache=false
   serviceName="zookeeper"
   principal="zookeeper/kerberos.example.com@EXAMPLE.COM";
};
```
## Start All Services
| *Service*  | *Command*   | 
|---|---|
|Zookeeper|`sudo -u zookeeper /opt/myapps/zookeeper-3.4.6/bin/zkServer.sh start  `|
|Zookeeper|`sudo -u storm /opt/myapps/apache-storm-1.1.1/bin/storm nimbus  `|
|Zookeeper|`sudo -u /opt/myapps/apache-storm-1.1.1/bin/storm supervisor `|
|Zookeeper|`sudo -u storm /opt/myapps/apache-storm-1.1.1/bin/storm ui `|
