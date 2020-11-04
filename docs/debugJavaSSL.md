# Debug java SSL for Keycloak under docker

## Context: 
Java app running under docker using registry.access.redhat.com/ubi8-minimal base image.  
Keycloak has a truststore and a keystore (for 2 way SSL / client certificate).
Keystore has for the same client/Org 1 sandbox and one prod certs in the keystore.
When Keycloak communicates with 3 party IDP Sandbox, it chooses the prod client cert.

```
<spi name="connectionsHttpClient">
<provider name="default" enabled="true">
    <properties>
        <property name="client-keystore" value="${jboss.home.dir}/certs/clientkeystore.jks"/>
        <property name="client-keystore-password" value="..."/>
        <property name="client-key-password" value="..."/>
    </properties>
</provider>
</spi>
<spi name="truststore">
<provider name="file" enabled="true">
    <properties>
        <property name="file" value="${jboss.home.dir}/certs/keystore.jks"/>
        <property name="password" value="..."/>
        <property name="hostname-verification-policy" value="WILDCARD"/>
        <property name="disabled" value="false"/>
    </properties>
</provider>
</spi>
```


## Options:
- Enable java flags and logging levels, and check logs
- Use tcpdump on container traffic and check flow with Wireshark

### Option1: java flags and logging levels
- Logging  
    - Add `<console-handler name="CONSOLE"><level name="DEBUG"/>` in config  
    - Add `<root-logger> <level name="DEBUG"/>` in config  
    - Add `export JAVA_OPTS="$JAVA_OPTS -Djavax.net.debug=all` in docker entry point file  
- Logs should have details like `javax.net.ssl|DEBUG|ClientHello.java:653|Produced ClientHello handshake message `, or `javax.net.ssl||SunX509KeyManagerImpl.java:392|matching alias: alias1@dom.com`
- FLow is readable
- Certificate request -> certificate authorities is encrypted


### Option2: Use tcpdump
- use registry.access.redhat.com/ubi8 instead registry.access.redhat.com/ubi8-minimal base image because minimal does not have yum, libcap ..
- install tcpdump using rpm because yum can not do it:
``` 
RUN curl -fL http://mirror.centos.org/centos/8/AppStream/x86_64/os/Packages/tcpdump-4.9.2-6.el8.x86_64.rpm -o /usr/local/bin/tcpdump-4.9.2-6.el8.x86_64.rpm \
&& chmod +x /usr/local/bin/tcpdump-4.9.2-6.el8.x86_64.rpm

RUN rpm -ivh usr/local/bin/tcpdump-4.9.2-6.el8.x86_64.rpm

RUN tcpdump  -D
```
- download jSSLKeyLog from https://jsslkeylog.github.io/ (agent that writes to a file the keylog needed by wireshark to decrypt)
- add to docker entry point `export JAVA_OPTS="$JAVA_OPTS -javaagent:${JBOSS_HOME}/path/jSSLKeyLog.jar=${JBOSS_HOME}/path/sslkl.log`
- start tcpdump `docker exec -u 0 -t -i 41351414141 tcpdump -i any -vv -w capture1.cap`
- perform your flow
- copy from docker container the tcpdump log and the keylog:
```
docker cp  41351414141:/opt/rh-sso/capture1.cap capture2.cap
docker cp  41351414141:/opt/rh-sso/certs/sslkl.log sslkl2.log
```
- open log in wireshark
- configure wireshark settings for TLS to use the keylog
- The result is that AppData is decrypted into readable format
- check the certificate request issued by server
- check the client certificate sends


## Links:
https://docs.microsoft.com/en-us/archive/blogs/nettracer/how-it-works-on-the-wire-iis-http-client-certificate-authentication
https://docs.apigee.com/api-platform/troubleshoot/runtime/ssl-handshake-failures
https://help.mulesoft.com/s/article/How-to-capture-network-traffic-between-two-systems
https://blog.codecentric.de/en/2020/01/decrypt-tls-traffic-to-kafka-using-wireshark/
https://docs.oracle.com/javase/7/docs/technotes/guides/security/jsse/ReadDebug.html
https://help.mulesoft.com/s/article/How-to-Debug-SSL-TLS-Traffic-Using-jSSLKeyLog-TCPDUMP-and-Wireshark
https://jsslkeylog.github.io/