# Spring Profiles

When you build an application you need different configurations for different environments — local, dev, staging, production. Different database URLs, different credentials, different log levels. Spring Profiles handle exactly this.

---

## The problem without profiles

Without profiles you'd manually change config every time you switch environments. Easy to forget, easy to push production credentials to dev accidentally. Profiles make this automatic.

---

## Setting up profiles with application.properties

Create separate properties files for each environment:

```
src/main/resources/
├── application.properties          ← shared/default config
├── application-dev.properties      ← dev environment
├── application-prod.properties     ← production environment
├── application-test.properties     ← testing
```

`application-dev.properties`:
```properties
spring.datasource.url=jdbc:mysql://localhost:3306/myapp_dev
spring.datasource.username=root
spring.datasource.password=root
spring.jpa.show-sql=true
logging.level.root=DEBUG
```

`application-prod.properties`:
```properties
spring.datasource.url=jdbc:mysql://prod-server:3306/myapp_prod
spring.datasource.username=prod_user
spring.datasource.password=strong_password_here
spring.jpa.show-sql=false
logging.level.root=ERROR
```

`application.properties` (shared):
```properties
spring.application.name=myapp
server.port=8080
```

---

## Activating a profile

**Option 1 — in application.properties:**

```properties
spring.profiles.active=dev
```

**Option 2 — as a JVM argument when running:**

```bash
java -jar myapp.jar --spring.profiles.active=prod
```

**Option 3 — as an environment variable:**

```bash
export SPRING_PROFILES_ACTIVE=prod
java -jar myapp.jar
```

In production you almost always use Option 2 or 3 — you don't hardcode the profile in the file.

---

## @Profile — conditional beans

You can create beans that only load for specific profiles:

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        // H2 in-memory DB for local development
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        // real MySQL for production
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(System.getenv("DB_URL"));
        config.setUsername(System.getenv("DB_USER"));
        config.setPassword(System.getenv("DB_PASS"));
        return new HikariDataSource(config);
    }
}
```

Only the matching bean gets created — the other one doesn't exist in that environment.

---

## @Profile on a whole class

```java
@Service
@Profile("dev")
public class MockEmailService implements EmailService {
    // doesn't actually send emails, just logs
    public void send(String to, String subject) {
        System.out.println("Mock email to: " + to + " | Subject: " + subject);
    }
}

@Service
@Profile("prod")
public class RealEmailService implements EmailService {
    // actually sends emails via SMTP
    public void send(String to, String subject) {
        // real email sending logic
    }
}
```

In dev you get the mock, in prod you get the real thing. The rest of your code just uses `EmailService` — doesn't care which implementation it gets.

---

## application.yml — cleaner for multiple profiles

If you prefer YAML you can put all profiles in one file using `---` separator:

```yaml
spring:
  application:
    name: myapp

---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:mysql://localhost:3306/myapp_dev
    username: root
    password: root
  jpa:
    show-sql: true

---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:mysql://prod-server:3306/myapp_prod
    username: prod_user
    password: ${DB_PASSWORD}
  jpa:
    show-sql: false
```

`${DB_PASSWORD}` reads from environment variable — never hardcode production passwords in config files.

---

## Stuff I want to remember

**What is the default profile in Spring Boot?**

If no profile is active, Spring uses the `default` profile. Only `application.properties` is loaded. You can explicitly name a bean for default profile with `@Profile("default")`.

**Can you activate multiple profiles at once?**

Yes. `spring.profiles.active=dev,swagger` activates both. Useful when some config is environment-based (dev/prod) and some is feature-based (swagger enabled or not).

**Where should you set the active profile in production?**

Never in the code or properties file. Always as an environment variable or JVM argument at runtime. This keeps your config flexible and avoids accidentally pushing the wrong profile to production.

---

*Before I knew about profiles I was commenting and uncommenting database URLs every time I switched between local and testing. Profiles fixed that completely.*