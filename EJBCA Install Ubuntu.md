# Introduction
This manual has been written and tested using EJBCA version 6.10.1.2 running on JBoss EAP 7.2.0 using MariaDB 10.1 as database engine.
All information provided in this document is current as of the 16th of August 2018.

# Prerequisites
## Update apt-get sources
Please run the following commands prior to performing any of the following steps:

    sudo apt update
    sudo apt upgrade    

## Download necessary files
Please download the latest EJBCA release and JBoss EAP installer.
Unfortunately - to my knowledge - RedHat does not offer a static download link for the JBoss installer .JAR file.
Since Ubuntu Server usually does not have a GUI installed, you must circumvent this restriction by starting the download on your administrative desktop machine, copying the download link, then download using wget.

## Java
I do recommend installing the OpenJDK Java framework. 
To do so, run 

    sudo apt install openjdk-8-jdk openjdk-8-demo openjdk-8-doc openjdk-8-jre-headless openjdk-8-source 

or the command installing the latest stable OpenJDK release. Do note that during my testing, building of the EJBCA EAR file failed with OpenJDK >8 due to some missing Java classes.

## ANT
Install ant by running

    sudo apt install ant

## MariaDB
Install MariaDB by running

    sudo apt install mariadb-server

After installing MariaDB, no root password has been set on the server.
If you wish to change this, run:

    sudo mariadb -u root
    GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' IDENTIFIED BY '<YOUR PASSWORD>';
    FLUSH PRIVILEGES;

Finally, run the following to create a new database for EJBCA:

    sudo mariadb -u root
    CREATE DATABASE ejbca CHARACTER SET utf8 COLLATE utf8_general_ci;
    CREATE USER 'ejbca'@'localhost' IDENTIFIED BY '<PASSWORD>';
    GRANT ALL PRIVILEGES ON ejbca.* TO 'ejbca'@'localhost';
    FLUSH PRIVILEGES;

## JBoss EAP
Install JBoss EAP by opening the JAR installer using

    java -jar <Path to JBoss JAR>

After having installed JBoss, you must start it by running

    $JBOSS_HOME/bin/standalone.sh

When running without a GUI, please use the following command

    $JBOSS_HOME/bin/standalone.sh -bmanagement 0.0.0.0 -b 0.0.0.0

to allow the JBoss management interface to be reachable from outside of the linux server.
After having started JBoss, open the JBoss CLI using the following command:

    $JBOSS_HOME/bin/jboss_cli.sh -c

Thereafter, type the following commands:

    /socket-binding-group=standard-sockets/socket-binding=remoting:add(port="4447")
    /subsystem=undertow/server=default-server/http-listener=remoting:add(socket-binding=remoting)
    :reload

# EJBCA Configuration
EJBCA provides sample configurations using the .sample file extension.
The following configuration changes are made significantly easier when running 

    mv <file>.properties.sample <file>.properties

for each setting to be configured.

## certstore.properties
Add/edit the following line:

    certstore.enabled=true

## cesecore.properties
Add/edit the following lines:

    password.encryption.key=<Password Encryption Key>
    password.encryption.count=100
    ca.keystorepass=<KeyStore Password>
    ca.serialnumberoctetsize=16

## database.properties
As the driver name is not neccessarily easy to determine, I do recommend starting the DataStore creation wizard of the administrative web console of JBoss.
Add/edit the following lines:

    database.name=mysql
    database.url=jdbc:mysql://127.0.0.1:3306/ejbca?characterEncoding=UTF-8
    database.driver=mysql_com.mysql.jdbc.Driver_5_1
    database.username=ejbca
    database.password=<Ejbca User’s MySQL password>
    database.useSeparateCertificateTable=true

## ejbca.properties
Add/edit the following lines:

    appserver.home=<JBoss home directory>
    ejbca.productionmode=true
    ca.cmskeystorepass=<CMS Keystore Password>
    jboss.config=production
    ejbca.cli.defaultusername=ejbcacli
    ejbca.cli.defaultpassword=<CLI password>

## install.properties
Add/edit the following lines:

    ca.name=ManagementCA
    ca.dn=CN=ManagementCA,O=<Your Organisation>,C=<Your Country>
    ca.tokentype=soft
    ca.keyspec=secp521r1
    ca.keytype=ECDSA
    ca.signaturealgorithm=SHA512WithECDSA
    ca.validity=3650
    ca.policy=null

## mail.properties
If you want EJBCA to be able to send emails, add/edit the following lines:

    mail.user=<SMTP username>
    mail.password=<SMTP password>
    mail.smtp.host=<SMTP host>
    mail.smtp.port=<SMTP server port>
    mail.smtp.auth=true
    mail.smtp.starttls.enable=true
    mail.from=<EJBCA email address corresponding to <SMTP user>>
    mail.contentencoding=UTF-8

# EJBCA Installation
Open a new command line and navigate to $EJBCA_HOME.
Type 

    ant clean deploy

to build the EJBCA EAR and deploy it to JBoss.
After having deployed EJBCA successfully, stop the JBoss server and edit the following file:

    $JBOSS_HOME/standalone/configuration/standalone.xml

Find the line

    <http-connector name="http-remoting-connector" connector-ref="default" security-realm="ApplicationRealm"/>

and change it to

    <http-connector name="http-remoting-connector" connector-ref="remoting" security-realm="ApplicationRealm"/>

You must now **restart** the JBoss server.
To now install EJBCA and create the neccessary CAs, type

    ant runinstall

in the EJBCA_HOME directory.
Having successfully installed EJBCA, run

    ant deploy-keystore

to deploy the created keystores to JBoss.

# JBoss configuration
After the installation of EJBCA, we need to reconfigure JBoss to use the previously generated and deployed keystores for HTTPS.
To do so, firstly remove any old TLS and HTTP configuration:

    /subsystem=undertow/server=default-server/http-listener=default:remove
    /subsystem=undertow/server=default-server/https-listener=https:remove
    /socket-binding-group=standard-sockets/socket-binding=http:remove
    /socket-binding-group=standard-sockets/socket-binding=https:remove
    :reload

When JBoss has reloaded, add the required interfaces and Socket Bindings as follows:

    /interface=http:add(inet-address="0.0.0.0")
    /interface=httpspub:add(inet-address="0.0.0.0")
    /interface=httpspriv:add(inet-address="0.0.0.0")
    /socket-binding-group=standard-sockets/socket-binding=http:add(port="8080",interface="http")
    /socket-binding-group=standard-sockets/socket-binding=httpspriv:add(port="8443",interface="httpspriv")
    /socket-binding-group=standard-sockets/socket-binding=httpspub:add(port="8442", interface="httpspub")
    /subsystem=undertow/server=default-server/http-listener=http:add(socket-binding=http)
    /subsystem=undertow/server=default-server/http-listener=http:write-attribute(name=redirect-socket, value="httpspriv")
    :reload

As soon as JBoss has reloaded, we can now add and configure the new SSL security realms and configure them to use the keystores generated during EJBCAs installation:

    /core-service=management/security-realm=SSLRealm:add()
    /core-service=management/security-realm=SSLRealm/server-identity=ssl:add(keystore-path="${jboss.server.config.dir}/keystore/keystore.jks", keystore-password="<Keystore Password>", alias="localhost")
    /core-service=management/security-realm=SSLRealm/authentication=truststore:add(keystore-path="${jboss.server.config.dir}/keystore/truststore.jks", keystore-password="<Truststore Password>")
    
**Shutdown** the server and restart it to ready it for the next set of commands:

    /subsystem=undertow/server=default-server/https-listener=httpspriv:add(socket-binding=httpspriv, security-realm="SSLRealm", verify-client=REQUIRED)
    /subsystem=undertow/server=default-server/https-listener=httpspriv:write-attribute(name=max-parameters, value="2048")
    /subsystem=undertow/server=default-server/https-listener=httpspub:add(socket-binding=httpspub, security-realm="SSLRealm")
    /subsystem=undertow/server=default-server/https-listener=httpspub:write-attribute(name=max-parameters, value="2048")
    :reload

You have now configured the HTTP and HTTPS bindings for JBoss.
Finally, add a few system properties required for things like OCSP requests:

    /system-property=org.apache.tomcat.util.buf.UDecoder.ALLOW_ENCODED_SLASH:add(value=true)
    /system-property=org.apache.catalina.connector.CoyoteAdapter.ALLOW_BACKSLASH:add(value=true)
    /system-property=org.apache.catalina.connector.URI_ENCODING:add(value="UTF-8")
    /system-property=org.apache.catalina.connector.USE_BODY_ENCODING_FOR_QUERY_STRING:add(value=true)
    /subsystem=webservices:write-attribute(name=wsdl-host, value=jbossws.undefined.host)
    /subsystem=webservices:write-attribute(name=modify-wsdl-address, value=true)
    :reload

# Accessing EJBCA
To access EJBCAs administrative section of the web interface, you have to install the SuperAdmin certificate, stored at $EJBCA_HOME/p12/superadmin.p12, which has been generated during the ant runinstall command.

The EJBCA web interface may be reached at 

    https://<serverip>:8442/ejbca

