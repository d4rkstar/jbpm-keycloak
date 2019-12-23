# Integration between JBPM and keycloak

## Prerequisites

1. A working keycloak installation with admin access
2. Some patience and :coffee:

## References

- [JBPM's Keycloak SSO integration](https://docs.jboss.org/jbpm/release/7.31.0.Final/jbpm-docs/html_single/#_kie.keycloakssointegration)
- [Uberfire security management](https://github.com/kiegroup/appformer/tree/master/uberfire-extensions/uberfire-security/uberfire-security-management/uberfire-security-management-keycloak)
- [Secure spring-boot 2 using Keycloak](https://medium.com/keycloak/secure-spring-boot-2-using-keycloak-f755bc255b68)


## Steps

### Keycloak

Inside keycloak, create a new realm. Call it as you want. In our demo, we'll use a realm called "jbpm".

### KIE

- download a business application stub from https://start.jbpm.org. Then unzip the folder. 

```bash
$ unzip business-application.zip -d bapp
```

:point_right: [Further informartions here](https://docs.jboss.org/jbpm/release/7.31.0.Final/jbpm-docs/html_single/#_create_your_business_application) :point_left:

You'll end with three folders:

- *business-application-kjar* - the process
- *business-application-model* - model and shared data
- *business-application-service* - the kie server service

Keycloak integration is achieved changing some things inside the service as described [here](https://docs.jboss.org/jbpm/release/7.31.0.Final/jbpm-docs/html_single/#_use_keycloak_as_authentication_provider).

In particular:

1. Configure the keycloak client and do the other things related to keycloak
2. Configure the pom.xml adding dependencies. Mine is like this:

```xml
[...]

<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.1.RELEASE</version>
  </parent>

  <properties>
    <version.org.kie>7.30.0.Final</version.org.kie>
    <version.org.keycloak>8.0.1</version.org.keycloak>
    <maven.compiler.target>1.8</maven.compiler.target>
    <maven.compiler.source>1.8</maven.compiler.source>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <narayana.version>5.9.0.Final</narayana.version>
    <fabric8.version>3.5.40</fabric8.version>
  </properties>

  <dependencies>
    <dependency>
        <groupId>org.kie</groupId>
        <artifactId>kie-server-spring-boot-starter</artifactId>
        <version>${version.org.kie}</version>
    </dependency>

    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>

    <dependency>
      <groupId>org.keycloak</groupId>
      <artifactId>keycloak-spring-boot-starter</artifactId>
    </dependency>
  </dependencies>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.keycloak.bom</groupId>
        <artifactId>keycloak-adapter-bom</artifactId>
        <version>${version.org.keycloak}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>

    </dependencies>
  </dependencyManagement>

  <build>
  [....]
```

3. Configure the application.properties file inside the file ```bapp/business-application-service/src/main/resources/application.properties```. This file is named "-dev.properties" for the dev environment (when running launch-dev.sh or java -jar -Dspring.profiles.active=dev -jar jarname).

Significative part of the configuration are:

:warning: *Attention* :warning:
If you're using the kieserver through a controller (like our case with Business Central):
```
#kie server config
kieserver.serverId=business-application-service-dev-bs
kieserver.serverName=business-application-service Dev BS
kieserver.location=http://localhost:8090/rest/server
kieserver.controllers=http://localhost:8080/business-central/rest/controller
```

This part is for keycloak
```
# keycloak security setup
keycloak.auth-server-url=http://keycloak-public-url-or-ip:8180/auth
keycloak.realm=demo
keycloak.resource=springboot-app
keycloak.public-client=true
keycloak.principal-attribute=preferred_username
keycloak.enable-basic-auth=true
```

4. We need to change some java code inside the service. In particular, we need to change the [```DefaultWebSecurityConfig.java```](https://gist.github.com/d4rkstar/2f036294aa20612ba7a3e6c888e87ca4#file-defaultwebsecurityconfig-java) inside the ```src/main/java/com/company/service``` folder and add the file. In the same folder we need to add the file [```CustomKeycloakSpringBootConfigResolver.java```](https://gist.github.com/d4rkstar/2f036294aa20612ba7a3e6c888e87ca4#file-customkeycloakspringbootconfigresolver-java) [reference here](https://medium.com/keycloak/secure-spring-boot-2-using-keycloak-f755bc255b68)

You can find both file on this [Public Gist](https://gist.github.com/d4rkstar/2f036294aa20612ba7a3e6c888e87ca4)


Usually, the controller (the business-central application in our case) requires credentials on the ```rest/controller``` endpoint. Defaults credentials used by uberfire security client are kieserver/kieserver1! 

To change these credentials, we need to start our kie server passing user/pwd (look inside [Jboss Doc chapter 22, table 22.1](https://docs.jboss.org/drools/release/6.4.0.CR2/drools-docs/html/ch22.html)):

- org.kie.server.controller.user - string (default is "kieserver") Username used to connect to the controller REST api
- org.kie.server.controller.pwd - string (default is "kieserver1!") Password used to connect to the controller REST api

We can change these in two ways:

1. from command line, passing in parameters to the jar file:

```bash
$ java -Dspring.profiles.active=dev -Dorg.kie.server.controller.user=kieserver -Dorg.kie.server.controller.pwd=kieserver -jar target/business-application-service-1.0-SNAPSHOT.jar
```

2. configuring the ```business-application-service.xml``` (or the dev file). Ex.

```xml
<kie-server-state>
  <controllers>
    <string>http://localhost:8080/business-central/rest/controller</string>
  </controllers>
  <configuration>
    <configItems>
      <config-item>
        <name>org.kie.server.controller.user</name>
        <value>kieserver</value>
        <type>java.lang.String</type>
      </config-item>
      <config-item>
        <name>org.kie.server.controller.pwd</name>
        <value>kieserver1!</value>
        <type>java.lang.String</type>
      </config-item>
      <config-item>
        <name>org.kie.server.location</name>
        <value>http://localhost:8090/rest/server</value>
        <type>java.lang.String</type>
      </config-item>
      <config-item>
        <name>org.kie.server.id</name>
        <value>business-application-service-dev</value>
        <type>java.lang.String</type>
      </config-item>
      <config-item>
        <name>org.kie.server.controller</name>
        <value>http://localhost:8080/business-central/rest/controller</value>
        <type>java.lang.String</type>
      </config-item>
    </configItems>
  </configuration>
  <containers/>
</kie-server-state>
```

### JBPM Downloads

- download jbpm from https://jbpm.org. actual release is 7.31.0.Final

```bash
$ wget https://download.jboss.org/jbpm/release/7.31.0.Final/jbpm-server-7.31.0.Final-dist.zip
```

- unzip the archive to a predefined folder:

```bash
$ unzip jbpm-server-7.31.0.Final-dist.zip -d jbpm 
```

- go inside the folder ```jbpm/standalone/deployments``` and remove the following jar files:

```bash
$ cd jbpm/standalone/deployments
$ rm -rf kie-server.war*
$ rm -rf jbpm-casemgmt.war*
```

you should end with a directory with the only Business Central war file inside.

We'll come back here later.



