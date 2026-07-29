# Vprofile

A Java Spring MVC web application (WAR packaging) demonstrating a multi-tier architecture, built with Maven and deployed via a CI/CD pipeline to AWS with GitOps-based delivery through Helm.

## Architecture

The application is a multi-tier system with the following components:

| Component     | Technology         | Purpose                          |
|----------------|--------------------|-----------------------------------|
| App server     | Tomcat 10 (JDK 21) | Hosts the Spring MVC WAR          |
| Database       | MySQL 8.0          | Persistent storage (`accounts` DB)|
| Cache          | Memcached          | Active/standby caching layer      |
| Message queue  | RabbitMQ           | Async messaging                   |
| Search         | Elasticsearch      | Search indexing                   |
| Load balancer  | Nginx              | Reverse proxy in front of Tomcat  |

Service hostnames (`vprodb`, `vprocache01`, `vpromq01`) are configured in [src/main/resources/application.properties](src/main/resources/application.properties) and are expected to be resolvable on the network the app runs in (e.g. via Docker Compose service names or Kubernetes service DNS).

## Tech stack

- Java 21, Spring 6 / Spring Security / Spring Data JPA, Hibernate
- Maven (WAR packaging)
- JSP views (`/WEB-INF/views`)
- Docker (multi-stage builds)
- SonarQube (self-hosted) + Checkstyle for code quality
- Amazon ECR for image storage, Helm for Kubernetes deployment manifests

## Project structure

```
.
├── src/main/java          # Application source (com.visualpathit package)
├── src/main/resources     # application.properties, logback config, SQL seed data
├── src/main/webapp         # JSPs, static resources, Spring XML config (WEB-INF)
├── src/test                # Unit tests
├── Docker-files/
│   ├── app/                # Tomcat image (expects a pre-built WAR)
│   │   └── multistage/     # Multi-stage build (Maven build + Tomcat runtime)
│   ├── web/                # Nginx reverse proxy image
│   └── db/                 # MySQL image with seed data
├── .github/workflows/ci.yml # CI/CD pipeline (see below)
├── sonar-project.properties # SonarQube scan configuration
└── pom.xml
```

## Building locally

Requires JDK 21 and Maven.

```bash
mvn clean package
```

This produces `target/vprofile-v2.war`. Run unit tests only with:

```bash
mvn test
```

Run a static Checkstyle report with:

```bash
mvn checkstyle:checkstyle
```

To run the app locally without Docker (via the Jetty plugin, pointing at your own MySQL/Memcached/RabbitMQ instances):

```bash
mvn jetty:run
```

## Running with Docker

Build the Tomcat image from a WAR you've already built with `mvn package`:

```bash
docker build -f Docker-files/app/Dockerfile -t vprofileapp .
```

The database and web (Nginx) images can be built similarly from `Docker-files/db/Dockerfile` and `Docker-files/web/Dockerfile`. All three services need to be able to reach each other on the hostnames configured in `application.properties`.

> **Note:** `Docker-files/app/multistage/Dockerfile` currently clones `hkhcoder/vprofile-project` from GitHub rather than building from this repository's own source — it's a placeholder from the original course material and does not yet reflect local changes if used as-is.

## CI/CD pipeline

Defined in [.github/workflows/ci.yml](.github/workflows/ci.yml):

1. **Feature branch push** — no pipeline runs.
2. **Pull request to `main`** — Maven build, unit tests, Checkstyle, SonarQube scan, and SonarQube quality gate check (self-hosted SonarQube on EC2). A failing quality gate blocks the check; merge protection on `main` should require this check to pass.
3. **Merge to `main`** — builds the Docker image from `Docker-files/app/multistage/Dockerfile`, pushes it to Amazon ECR (`vprofileappimg`, `ap-south-2`) tagged with both the commit SHA and `latest`, then updates `app.image` / `app.tag` in the separate `vprofile-helm` repo's `helm/vprofile/values.yaml` using `yq`, committing the change so a GitOps controller (e.g. Argo CD / Flux) can pick it up.

### Required GitHub configuration

**Secrets:** `SONAR_TOKEN`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `HELM_REPO_USER`, `GITOPS_PAT`

**Variables:** `AWS_REGION`, `ECR_REPOSITORY`, `HELM_REPO_NAME`, `SONAR_HOST_URL`

## Code quality

SonarQube project settings live in [sonar-project.properties](sonar-project.properties). Checkstyle results and (if enabled) JaCoCo coverage are fed into the same Sonar analysis.
