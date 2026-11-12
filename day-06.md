# Day 6: Jenkins + Maven Pipeline

**Date:** 12 Nov 2026 | **Region:** Mumbai (ap-south-1) | **Status:** Completed

## Goal

Put a Spring Boot app on GitHub, write a `Jenkinsfile` with checkout, build and test stages, and get a green pipeline in Jenkins. Then break Maven on purpose and fix it.

## The idea in simple words

| Piece | Think of it as |
|---|---|
| Spring Boot app | The product being built (a Java web application) |
| Maven (`mvn`) | The tool that downloads libraries, compiles, tests and packages the app into a `.jar` |
| Jenkinsfile | The assembly-line recipe, stored in the repo next to the code |
| Stage | One step of the recipe (Checkout, Build, Test) |

**Why the Jenkinsfile lives in the repo:** the pipeline becomes code. It is version-controlled (I can see who changed a step and roll it back), it travels with the app and its branches, and if the Jenkins server is lost, the recipe is still in GitHub.

## Environment

| Item | Value |
|---|---|
| Server | `84DayChallenge`, t3.small (2 vCPU, 1.9 GiB RAM), Ubuntu 26.04 LTS |
| Java | OpenJDK 21 **JDK** (the compiler is needed, not only the JRE) |
| Maven | 3.9.12 (from apt) |
| Jenkins | 2.580.2 |
| Repo | `jenkins-maven-demo` on GitHub |
| Job | Pipeline job, "Pipeline script from SCM", branch `main` |

## Steps

### 1. Tools on the server
```bash
sudo apt install -y openjdk-21-jdk maven git
javac -version      # 21.x
mvn -version        # Maven 3.9.12, Java 21 (must match javac)
git --version
```
Jenkins only needed the JRE on Day 5. Compiling the app needs the **JDK**.

### 2. Generate the app and build it by hand first
```bash
curl -s https://start.spring.io/starter.zip \
  -d type=maven-project -d language=java -d javaVersion=21 \
  -d groupId=com.example -d artifactId=demo -d name=demo \
  -d dependencies=web -o demo.zip
unzip demo.zip -d demo-app && cd demo-app
mvn -B package
ls target/*.jar
```
Result: `BUILD SUCCESS` in about 27 seconds, with `target/demo-0.0.1-SNAPSHOT.jar` created. Memory after the build was still 1.1 GiB available with Jenkins running, so no swap was needed.

**Rule:** always run the build by hand once. If it works by hand, a later Jenkins failure is a Jenkins problem.

### 3. Push to GitHub
```bash
git init -b main
git add . && git commit -m "Initial Spring Boot demo app"
git remote add origin https://github.com/<username>/jenkins-maven-demo.git
git push -u origin main
```
GitHub does not accept account passwords for `git push`. I used a **fine-grained personal access token** (one repo, Contents: read and write, 7-day expiry). The token is never put in notes or the repo.

### 4. The Jenkinsfile
```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn -B test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
    }
}
```
- Build uses `-DskipTests` so a failure tells me **which stage** broke.
- `junit` publishes test results, and `always` runs it even when tests fail.

### 5. The pipeline job
New Item > Pipeline > Pipeline script from SCM > Git > repo URL > branch `main` > script path `Jenkinsfile`. Then **Build Now**.

## Result

- Checkout pulled the latest commit from GitHub.
- Build and Test stages passed: `Tests run: 1, Failures: 0, Errors: 0`.
- `Finished: SUCCESS`. This is the green pipeline from a Jenkinsfile.
- `[Checks API] No suitable checks publisher found` is harmless. It only means no GitHub Checks plugin is configured.

## Troubleshooting drill: `mvn: command not found`

I hid Maven on purpose:
```bash
sudo mv /usr/bin/mvn /usr/bin/mvn.off
```

The **Build** stage failed:
```
+ mvn -B -DskipTests clean package
mvn: not found
ERROR: script returned exit code 127
Finished: FAILURE
```

What it showed:
- **Exit code 127 means "command not found"** in Linux.
- The **Test** stage was skipped: Jenkins stops at the first failing stage.
- The build runs as the **`jenkins` user** in `/var/lib/jenkins/workspace/...`, so its PATH can differ from my own terminal's. This is the same bug as the Day 1 cron drill.

**Fix:** put Maven back on the PATH (`sudo mv /usr/bin/mvn.off /usr/bin/mvn`) and run the build again. It went green.

Other fixes that also work:
- Set the PATH inside the Jenkinsfile:
  ```groovy
  environment { PATH = "/usr/share/maven/bin:${env.PATH}" }
  ```
- Configure Maven in **Manage Jenkins > Tools** and use a `tools { maven '...' }` block.

| Clue | Meaning |
|---|---|
| `command not found`, exit 127 | The tool is not on the PATH of the user running the build |
| Works in my terminal, fails in Jenkins | Jenkins runs as the `jenkins` user with its own environment |
| Next stage "skipped due to earlier failure" | Normal: a failed stage stops the pipeline |

## Key takeaways

- Build by hand first, then automate it.
- A JDK is needed to compile, and a JRE is only enough to run.
- Keep `javac` and `mvn` on the same Java version to avoid confusing errors.
- Read the console output top-down. The first red stage is the cause, and later stages are only skipped.
- Never paste a GitHub token into notes, chat or a repo.

## Cleanup

- Deleted nothing in AWS. Stopped `84DayChallenge` (stopped, not terminated).
- The GitHub token expires in 7 days, and I will delete it after Day 7's webhook work.

## Not done

- Maven in Global Tool Configuration (the Maven Integration plugin was not set up). Fixing the PATH was enough for the drill.
