# Spring Boot Integration with Liquibase

Spring Boot makes integrating Liquibase incredibly easy. You don't need to run a separate command-line tool to execute migrations; the framework handles it during application startup.

## How it Works
1. **Dependency:** You add the Liquibase dependency to your `pom.xml` (Maven) or `build.gradle` (Gradle).
2. **Auto-Configuration:** Spring Boot detects Liquibase on the classpath.
3. **Execution Phase:** Before Spring Boot creates the `ApplicationContext` (before your `@Service` or `@Repository` beans are fully initialized), it runs Liquibase.
4. **Result:** This ensures that when your application code starts interacting with the database, the schema is 100% up-to-date and matches the code's expectations.

## Maven Dependency
To enable Liquibase, include this dependency in your `pom.xml`:

```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

## Basic Configuration (application.yml)
Spring Boot exposes several properties to configure Liquibase via `application.yml` or `application.properties`.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/myapp
    username: myuser
    password: mypassword
    driver-class-name: org.postgresql.Driver

  liquibase:
    enabled: true
    change-log: classpath:/db/changelog/db.changelog-master.yaml # Default path
    contexts: dev, local # Only run ChangeSets with these contexts
    drop-first: false # DANGEROUS: If true, it drops the entire database schema before running! Useful only for local testing.
    default-schema: public # Specify the schema if different from the default
```

## Default Directory Structure
By default, Spring Boot expects to find your main changelog file at:
`src/main/resources/db/changelog/db.changelog-master.yaml` (or `.xml`, `.json`, `.sql`).

A common, clean pattern:
```text
src/main/resources/
└── db/
    └── changelog/
        ├── db.changelog-master.yaml
        ├── changes/
        │   ├── 001-initial-schema.yaml
        │   ├── 002-add-user-roles.yaml
        │   └── 003-seed-reference-data.yaml
        └── rollbacks/
            └── custom-rollback-scripts.sql
```

## Liquibase and Spring Data JPA / Hibernate
If you use Hibernate (via Spring Data JPA), you might be familiar with:
`spring.jpa.hibernate.ddl-auto=update`

**CRITICAL RULE:** When using Liquibase, you must **never** use `update` or `create-drop` in production. 

You should set it to `validate` or `none`:
```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
```
This tells Hibernate: "I will generate my tables using Liquibase. When you start, just check if the tables match your `@Entity` classes. Do not try to modify the schema yourself."

## Profiles and Contexts
You can wire Spring Boot profiles to Liquibase contexts.

`application-dev.yml`:
```yaml
spring:
  liquibase:
    contexts: dev
```

`application-prod.yml`:
```yaml
spring:
  liquibase:
    contexts: prod
```
This allows you to insert massive test datasets during development but exclude them entirely when deploying to production.
