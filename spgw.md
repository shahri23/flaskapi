To configure mutual TLS (mTLS) with backend services in Spring Cloud Gateway using only properties.yaml (application.yaml), you can leverage Spring Boot's auto-configuration capabilities along with custom properties. Here's how you can achieve this:

Configure SSL/TLS for Spring Cloud Gateway:

You can configure SSL/TLS for Spring Cloud Gateway using properties in the application.yaml file. Specify the keystore and truststore paths along with other SSL-related properties.

```yaml
server:
  port: 8443
  ssl:
    key-store: classpath:keystore.p12
    key-store-password: changeit
    key-store-type: PKCS12
    trust-store: classpath:truststore.p12
    trust-store-password: changeit
    trust-store-type: PKCS12
    client-auth: need
```

Configure Route for Backend Service:

Define routes in the application.yaml file to specify the backend services along with any additional configuration required for mTLS.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: backend_route
          uri: https://backend.example.com
          predicates:
            - Path=/backend/**
          filters:
            - PrefixPath=/api
```

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

Ensure Backend Service Requires mTLS:

Make sure your backend service is configured to require client certificates for authentication.

Testing and Debugging:

Test your setup to ensure mutual TLS authentication is working correctly between the gateway and backend services. Debug any issues that may arise during the configuration process.

With the above configuration, Spring Cloud Gateway should automatically handle mTLS authentication with the backend services without requiring any Java code changes. Make sure to replace the placeholder values (keystore.p12, truststore.p12, https://backend.example.com, etc.) with the actual values relevant to your setup.
