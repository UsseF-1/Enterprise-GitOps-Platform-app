# Visualpathit VProfile Webapp

## Overview

This project is a Java web application built with Spring MVC, Spring Security, Spring Data JPA, Hibernate, and Jakarta EE technologies. It is packaged as a WAR and intended to run on a servlet container or as part of a Jetty-powered deployment.

The application includes:
- User authentication and authorization
- JDBC-based MySQL persistence
- RabbitMQ messaging support
- Elasticsearch integration
- Memcached caching support
- JSP-based web UI under `/WEB-INF/views`

## Project Structure

- `pom.xml` - Maven build configuration and dependencies
- `src/main/java` - Java source code
- `src/main/resources` - application properties and SQL resources
- `src/main/webapp` - web assets, JSP views, and configuration files
- `src/test/java` - unit and integration tests

## Requirements

- Java 17
- Maven 3.8+ (or compatible)
- MySQL database
- RabbitMQ broker
- Elasticsearch cluster/node
- Memcached cluster (optional depending on runtime use)
- Servlet container or Jetty Maven plugin for local execution

## Configuration

Application configuration is driven by `src/main/resources/application.properties`.

Important configuration values include:
- `jdbc.url` - MySQL JDBC connection string
- `jdbc.username` / `jdbc.password` - MySQL credentials
- `rabbitmq.address`, `rabbitmq.port`, `rabbitmq.username`, `rabbitmq.password`
- `elasticsearch.host`, `elasticsearch.port`, `elasticsearch.cluster`, `elasticsearch.node`
- `memcached.active.host`, `memcached.active.port`, `memcached.standBy.host`, `memcached.standBy.port`

View resolver settings:
- `spring.mvc.view.prefix=/WEB-INF/views/`
- `spring.mvc.view.suffix=.jsp`

Multipart upload limits:
- `spring.servlet.multipart.max-file-size=128KB`
- `spring.servlet.multipart.max-request-size=128KB`

## Build

From the project root, run:

```bash
mvn clean package
```

This produces a WAR artifact in `target/`.

## Run Locally

### Using Jetty

Run the application locally with Maven Jetty:

```bash
mvn jetty:run
```

Then open the application in a browser at `http://localhost:8080/`.

### Deploying to a Servlet Container

Build the WAR and deploy `target/vprofile-v2.war` to any compatible servlet container that supports Jakarta EE 10 / Servlet 6.

## Testing

Run unit tests with:

```bash
mvn test
```

## Notes

- The Maven configuration includes Spring Framework 6, Spring Boot test artifacts, Spring Security 6, and modern Jakarta APIs.
- Elasticsearch dependencies target version 7.10.2; ensure the runtime cluster is compatible.
- `jakarta.servlet-api` and `jakarta.jakartaee-api` are marked as provided, so the runtime container must supply servlet/Jakarta EE APIs.

## Contact

If you need help with this project, inspect `src/main/resources/application.properties`, the Spring configuration XML files under `src/main/webapp/WEB-INF/`, and the Java packages under `com.visualpathit.account`.
