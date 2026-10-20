# TP 02 — GitHub Actions

Repository: https://github.com/YoussefAG1337/tp-devops-correction-docker

The default branch is `main`. References to `master` in the handout therefore
mean `main` in this repository. Each image retains its own folder:
`simple-api/`, `database/` and `http-server/`.

## Checkpoint 1 — Build and test the backend

`.github/workflows/main.yml` runs on pushes to `main` and `develop`, on pull
requests and on manual dispatch. The `test-backend` job uses Ubuntu 24.04,
Temurin JDK 21 and a Maven dependency cache. Its command is:

```sh
mvn --batch-mode --no-transfer-progress clean verify --file simple-api/pom.xml
```

Surefire runs unit tests. Failsafe is explicitly bound to `integration-test`
and `verify` so the controller tests named `*IT` also execute. The runner's
Docker daemon supplies an isolated PostgreSQL container. No shared database
or production database password is required. JaCoCo generates coverage at
`verify`, after both test suites. Reports are retained as the
`backend-test-reports` Actions artifact, including when a test fails.

### 2-1 — What are Testcontainers?

Testcontainers are libraries for starting disposable Docker containers during
automated tests. This application uses the JDBC and PostgreSQL modules: the
`jdbc:tc:postgresql:14.1-alpine:///simple-api` URL creates a real PostgreSQL
instance for the integration tests. Containers are cleaned up after testing.
This tests actual SQL and database behavior without sharing mutable test data
between pipeline runs.
