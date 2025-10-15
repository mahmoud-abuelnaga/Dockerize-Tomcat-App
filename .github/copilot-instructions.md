# Copilot instructions for Dockerize-Tomcat-App

These notes make AI coding agents immediately productive in this repo by capturing the actual architecture, workflows, and project-specific conventions.

## Big picture

- Stack is orchestrated with Docker Compose (`compose.yaml`). Services:
  - backend: Tomcat running a classic Spring MVC WAR built from `app/`
  - database: MariaDB seeded by `config/db/init.sql` (+ `.env`)
  - cache: Memcached
  - message-broker: RabbitMQ (+ `.env`)
  - reverse-proxy: NGINX proxying `backend:8080` to `localhost:8080`
- Source of truth for the webapp lives in `app/`:
  - Maven WAR project (`app/pom.xml`) → builds `target/vprofile-<version>.war`
  - Multi-stage Docker build in `app/Dockerfile` → copies WAR to Tomcat as `ROOT.war`
  - Spring is configured with XML under `app/src/main/webapp/WEB-INF/appconfig-*.xml`

## Runtime and wiring (what talks to what)

- Spring root context (`appconfig-root.xml`) imports MVC, Data/JPA, RabbitMQ, and Security; component-scan base package `com.visualpathit.account.*`; `application.properties` is the single config source.
- Data/JPA (`appconfig-data.xml`):
  - Pool: `org.apache.commons.dbcp2.BasicDataSource` with `${jdbc.*}` from `application.properties`
  - JPA: `LocalContainerEntityManagerFactoryBean`, `HibernateJpaVendorAdapter`, repos under `com.visualpathit.account.repository`, `@Transactional` enabled
- MVC (`appconfig-mvc.xml`): annotation-driven controllers, static resources at `/resources/**`, JSP views via `InternalResourceViewResolver` with prefix `/WEB-INF/views/` and suffix `.jsp`
- Security (`appconfig-security.xml`):
  - Requires `ROLE_USER` for `/` and `/welcome`, login at `/login`, uses `com.visualpathit.account.service.UserDetailsServiceImpl` and `BCryptPasswordEncoder(strength=11)`, CSRF disabled
- RabbitMQ (`appconfig-rabbitmq.xml`): annotation-driven listeners, connection params from `${rabbitmq.*}`, a `SimpleRabbitListenerContainerFactory`
- External service hostnames match Compose service names in `application.properties`: `database`, `cache`, `message-broker`

## Build, run, and test (commands agents should use)

- Local Maven build (runs tests):
  - from `app/`: `mvn clean install`
- Dev server (optional, runs the WAR in-place):
  - from `app/`: `mvn org.eclipse.jetty:jetty-maven-plugin:run` (context path `/`)
- Container build/push workflow (uses Maven project.version):
  - from `app/`: `make docker-build` then `make docker-push`
- Full stack up via Compose (builds backend image from `app/Dockerfile`):
  - from repo root: `docker compose up -d`
  - browse: http://localhost:8080 (NGINX → Tomcat)

## Versioning and image build gotcha

- The WAR file name is `vprofile-${project.version}.war` (from `app/pom.xml`).
- `app/Dockerfile` copies `vprofile-${ARG version}.war` with default `ARG version="v2"`.
- When you bump `<version>` in `pom.xml`, also pass the matching build-arg or use the Makefile (which injects the computed version). If building with Compose, consider adding `build.args.version` to `compose.yaml` to avoid mismatch.

## Conventions agents should follow

- New Spring components (controllers/services/repos) belong under `com.visualpathit.account.*` to be auto-discovered.
- New JSP views live in `app/src/main/webapp/WEB-INF/views/` and are rendered via the resolver configured in `appconfig-mvc.xml`.
- Database schema/data for local runs is initialized from `config/db/init.sql`. Update it if you need seed data; persistent data is stored in the `database-volume` Docker volume.
- Security rules are XML-driven; when adding new endpoints, update `appconfig-security.xml` intercept-urls accordingly.
- Logging is via Logback (`app/src/main/resources/logback.xml`); a few Spring debug flags are set in `application.properties`.

## Integrations and dependencies

- DB: JDBC URL points to `jdbc:mysql://database:3306/accounts` (MariaDB image); credentials come from `config/db/.env`.
- Cache: Memcached host `cache:11211` (`spymemcached` client). Code references read settings from `application.properties`.
- Messaging: RabbitMQ host `message-broker:5672`; Spring AMQP is set up with annotation listeners.
- Elasticsearch libraries and properties exist, but no XML wiring is present; treat ES as optional/legacy until explicitly used.

## File map (start points for agents)

- Compose: `compose.yaml`
- Webapp: `app/` (Maven/pom, Dockerfile, Makefile)
- Spring XML: `app/src/main/webapp/WEB-INF/appconfig-*.xml`, `web.xml`
- Config: `app/src/main/resources/application.properties`
- Reverse proxy: `config/reverse-proxy/nginx.conf`
- DB seed/env: `config/db/init.sql`, `config/db/.env`
- Tests live under `app/src/test/java/com/visualpathit/account/**` and are JUnit 4 style

If anything above is unclear or you need `compose.yaml` to pass build args automatically, let me know and I can update the files to match the intended workflow.
