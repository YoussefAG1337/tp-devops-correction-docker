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

## Checkpoint 2 — Build and publish Docker images

The first build checkpoint adds `build-and-push-docker-image` after
`test-backend` using `needs: test-backend`. A matrix builds the three
independent contexts with Buildx and `docker/build-push-action@v6`:

- `simple-api/` → `tp-devops-simple-api`
- `database/` → `tp-devops-database`
- `http-server/` → `tp-devops-httpd`

This intermediate checkpoint intentionally uses `push: false` to validate all
three images before enabling publication in the final delivery workflow.

### 2-2 — Why use secured variables?

GitHub Actions secrets store credentials separately from source code. They
let the workflow authenticate without committing a Docker Hub or SonarCloud
token to a public repository. Tokens can be restricted and rotated without a
code change. GitHub masks registered secrets in logs, but workflows must still
avoid printing them or sending them to untrusted pull request code. Public
configuration such as the Sonar organization and project key can use Actions
variables instead.

### 2-3 — Why does the Docker job need the backend test job?

`needs: test-backend` makes the image job wait until compilation, unit tests
and integration tests have passed. Without that dependency, jobs can run in
parallel, and an image could be published even when the tests subsequently
fail. The handout's `build-and-test-backend` name refers to the job actually
named `test-backend` in this repository.

### 2-4 — Why push Docker images?

Pushing uploads a built image to a registry so another machine, a deployment
pipeline or a teammate can pull and run that exact application without
rebuilding the source. `latest` identifies the latest published main build;
a separate commit-SHA tag provides traceability to a particular source revision.

## Checkpoint 3 — SonarCloud quality gate

In SonarQube Cloud (the service called SonarCloud in the handout), import the
GitHub fork into an organization and use `main` as the project's main branch.
Choose CI-based analysis with GitHub Actions and disable automatic analysis
so the same project is not analyzed using two competing methods.

Configure these settings in the GitHub repository under **Settings → Secrets
and variables → Actions**:

- Repository secret `DOCKERHUB_USERNAME`: the lowercase Docker Hub username.
- Repository secret `DOCKERHUB_TOKEN`: a Docker Hub access token permitted to push the three course images.
- Repository secret `SONAR_TOKEN`: a SonarCloud token permitted to analyze the project.
- Repository variable `SONAR_ORGANIZATION`: the exact Sonar organization key.
- Repository variable `SONAR_PROJECT_KEY`: the exact Sonar project key.

The public keys can be found in the SonarCloud project's GitHub Actions
setup tutorial. Do not commit tokens to a file or paste them into an issue.

After `clean verify`, the pipeline calls the pinned SonarScanner for Maven
`5.8.0.7211` using the compiled classes and combined JaCoCo XML coverage report.
The token is supplied only through the `SONAR_TOKEN` environment variable.
`sonar.qualitygate.wait=true` makes CI wait for the server's verdict and fail
if the quality gate fails; the processing timeout is 300 seconds. There is
no `continue-on-error` and a missing token or project key is a visible failure.

Tests run on both `main` and `develop`. To support the SonarCloud Free plan,
analysis runs on `main` and same-repository PRs targeting `main`; fork PRs and
pushes to `develop` run tests without exposing analysis credentials.

To reproduce analysis locally, first set `SONAR_TOKEN` in your environment
and use the same scanner goal after `mvn clean verify`, passing your public
organization and project keys. Test execution requires JDK 21, Maven and a
running Docker daemon.

Sources: [Maven Failsafe usage](https://maven.apache.org/surefire/maven-failsafe-plugin/usage.html),
[SonarCloud GitHub Actions setup](https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/ci-based-analysis/github-actions-for-sonarcloud),
[quality gate parameters](https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/analysis-parameters/parameters-not-settable-in-ui),
[SonarCloud subscription plans](https://docs.sonarsource.com/sonarqube-cloud/administering-sonarcloud/managing-subscription/subscription-plans).
