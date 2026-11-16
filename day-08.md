# Spring Boot CI with GitHub Actions

A Spring Boot application built and tested by a GitHub Actions pipeline. This repo is part of my DevOps/MLOps learning plan (Day 8: GitHub Actions basics). It replaces the Jenkins pipeline I built in Week 1 with an equivalent workflow.

<img width="1600" height="855" alt="image" src="https://github.com/user-attachments/assets/5b865351-569b-4668-a1ce-6771f747fe9d" />


## What the pipeline does

The workflow lives in [`.github/workflows/ci.yml`](.github/workflows/ci.yml).

| Job | Runs on | What it does |
|---|---|---|
| `build` | GitHub-hosted (`ubuntu-latest`) | Checks out the code, sets up Java, packages the app with Maven, and uploads the JAR as an artifact |
| `test` | GitHub-hosted (`ubuntu-latest`) | Runs after `build` succeeds and runs the unit tests with `mvn test` |
| `on-my-runner` | Self-hosted (my own VM) | Proves a job can run on my own machine by printing its hostname and user |

```
push / pull_request / manual run
            |
          build  ---->  app-jar artifact
            |
          test

on-my-runner  (self-hosted runner)
```

## Triggers

- **Push to `main`**: runs the full pipeline.
- **Pull request**: runs the full pipeline before merging.
- **Manual run** (`workflow_dispatch`): the **Run workflow** button in the Actions tab, with parameters.

## Parameters (manual runs only)

| Input | Type | Default | Purpose |
|---|---|---|---|
| `java_version` | dropdown (`21`, `17`) | `21` | JDK used to build and test |
| `run_tests` | checkbox | ticked | Untick to skip the `test` job |
| `environment` | dropdown (`dev`, `staging`) | `dev` | Environment label printed in the build log |

Automatic runs (push and PR) have no inputs, so they fall back to Java 21 and always run the tests. The test job's condition is:

```yaml
if: github.event_name != 'workflow_dispatch' || inputs.run_tests
```

This means: run the tests unless it is a manual run with the box unticked.

## Artifact

The `build` job uploads the packaged application as **`app-jar`** (about 17 MB, a Spring Boot fat JAR that includes its dependencies). To run it locally:

```bash
# Download app-jar from the run page, then:
unzip app-jar.zip
java -jar <your-app-name>.jar
```

Requires Java 21. The app starts on `http://localhost:8080`.

## Self-hosted runner

A self-hosted runner is registered under **Settings > Actions > Runners**. Jobs use it with `runs-on: self-hosted`.

Setup in short:

```bash
mkdir actions-runner && cd actions-runner
# download and extract the runner using the commands shown on the GitHub page
./config.sh --url https://github.com/<your-username>/<your-repo> --token <TOKEN>
./run.sh
```

Security note: do not leave a self-hosted runner attached to a public repository, because code from anyone's pull request could run on the machine. I stop the runner (`Ctrl+C`) and remove it after each session.

## Troubleshooting log

Problems I hit while building this, and how I fixed them.

### 1. Build failed on automatic runs but passed on a manual run

- **Symptom:** push and PR runs failed in the `build` job. A manual run with Java 21 passed.
- **Root cause:** the workflow fell back to Java 17 when no input was given, but the project needs Java 21.
- **Diagnosis:** read the failed step's log, checked the Java version in `pom.xml`, compared it with `setup-java`.
- **Fix:** changed the fallback and the dropdown default to `21`.
- **Prevention:** keep the version in one place, for example a workflow-level `env` variable.

### 2. Job stuck in "Queued"

- **Symptom:** a job with `runs-on: [self-hosted, gpu]` never started.
- **Root cause:** no runner had the `gpu` label. A runner that is offline causes the same symptom.
- **Diagnosis:** checked **Settings > Actions > Runners** for the runner's labels and online status, then compared them with `runs-on`.
- **Fix:** changed `runs-on` back to `self-hosted` and started the runner with `./run.sh`.
- **Prevention:** use labels that match a registered runner, and check that the runner is online before debugging the workflow.

## Jenkins vs GitHub Actions

| Jenkins | GitHub Actions |
|---|---|
| `Jenkinsfile` | YAML file in `.github/workflows/` |
| Stage | Job |
| Agent | Runner |
| Build parameters | `workflow_dispatch` inputs |
| Plugins | Marketplace actions (`uses:`) |
| Webhook setup | Built-in `push` / `pull_request` triggers |

## What I learned

- Every job starts on a fresh machine, so each job checks out the code and sets up Java itself.
- Artifacts are how a build's output survives after the job's machine is deleted.
- An empty input and `false` look the same in a condition, so automatic runs need an explicit check.
- Most pipeline failures are environment mismatches, so read the failing step's log before changing code.
