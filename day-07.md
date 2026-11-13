# Day 7 – Jenkins Advanced Pipeline & Slack Notifications

## 1. Learning Objectives

Today's goal was to understand how to build a more advanced Jenkins Declarative Pipeline and integrate Slack for automated build notifications.

By the end of this session, I learned how to:

- Configure Jenkins pipeline parameters.
- Control stage execution using conditional statements.
- Understand pipeline-level and stage-level `post` blocks.
- Publish test results using JUnit.
- Integrate Jenkins with Slack.
- Store Slack tokens securely using Jenkins Credentials.
- Send success and failure notifications.
- Test pipeline failures safely using Jenkins Replay.

---

## 2. Jenkins Pipeline Parameters

Parameters allow users to customize pipeline execution when starting a build.

### Boolean Parameter

```groovy
booleanParam(
    name: 'RUN_TESTS',
    defaultValue: true,
    description: 'Run the Test stage?'
)
```

This parameter controls whether the Test stage should execute.

### Choice Parameter

```groovy
choice(
    name: 'BUILD_GOAL',
    choices: ['package', 'verify'],
    description: 'Maven goal for the Build stage'
)
```

This allows users to select a Maven build goal.

### Accessing Parameters

```groovy
${params.BUILD_GOAL}
params.RUN_TESTS
```

- `params.BUILD_GOAL` retrieves the selected Maven goal.
- `params.RUN_TESTS` retrieves the Boolean parameter value.

---

## 3. Conditional Stage Execution

Jenkins provides the `when` directive to control whether a stage should execute.

```groovy
stage('Test') {
    when {
        expression {
            params.RUN_TESTS
        }
    }

    steps {
        sh 'mvn -B test'
    }
}
```

**What happens?**

- If `RUN_TESTS` is `true`, the Test stage executes.
- If `RUN_TESTS` is `false`, the Test stage is skipped.

This makes pipeline execution flexible.

---

## 4. JUnit Test Reporting

Jenkins can collect and display test results using the JUnit plugin.

```groovy
post {
    always {
        junit 'target/surefire-reports/*.xml'
    }
}
```

The `always` condition ensures Jenkins attempts to publish test reports regardless of the stage result.

Maven Surefire generates XML test reports in:

```text
target/surefire-reports/
```

---

## 5. Jenkins Post Conditions

The `post` section executes actions based on the pipeline or stage result.

Important conditions:

| Condition | Purpose |
|---|---|
| `always` | Executes regardless of the result |
| `success` | Executes when the pipeline succeeds |
| `failure` | Executes when the pipeline fails |
| `unstable` | Executes when the pipeline is unstable |
| `changed` | Executes when the result changes from the previous build |

Today's pipeline uses:

```groovy
post {
    always {
        echo "Final Pipeline Result: ${currentBuild.currentResult}"
    }

    success {
        echo 'SUCCESS: Jenkins pipeline passed!'
    }

    failure {
        echo 'FAILURE: Jenkins pipeline failed!'
    }
}
```

---

## 6. Jenkins and Slack Integration

### Objective

Send automated Slack notifications whenever a Jenkins pipeline succeeds or fails.

### Slack Setup

1. Created a Slack workspace named `Jenkins Notifications`.
2. Created a Slack channel named `#all-jenkins-notifications`.
3. Created a Slack app and enabled the `chat:write` OAuth scope.
4. Installed the app in the workspace.
5. Invited the bot into the notification channel.
6. Configured Jenkins Slack Notification plugin.
7. Stored the Slack Bot User Token in Jenkins Credentials.

### Secure Credential Configuration

In Jenkins:

**Manage Jenkins → Credentials → System → Global credentials**

Created a Secret Text credential:

| Setting | Value |
|---|---|
| Kind | Secret text |
| ID | `slack-token` |
| Description | Jenkins Slack Bot Token |
| Secret | Slack Bot User Token |

The actual token must remain private. It should never be hardcoded in the Jenkinsfile or committed to GitHub.

### Slack Success Notification

```groovy
slackSend(
    channel: '#all-jenkins-notifications',
    color: 'good',
    tokenCredentialId: 'slack-token',
    botUser: true,
    message: "SUCCESS: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}\nBuild URL: ${env.BUILD_URL}"
)
```

### Slack Failure Notification

```groovy
slackSend(
    channel: '#all-jenkins-notifications',
    color: 'danger',
    tokenCredentialId: 'slack-token',
    botUser: true,
    message: "FAILED: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}\nBuild URL: ${env.BUILD_URL}"
)
```

### Important Jenkins Environment Variables

| Variable | Meaning |
|---|---|
| `env.JOB_NAME` | Jenkins job name |
| `env.BUILD_NUMBER` | Current build number |
| `env.BUILD_URL` | URL of the current build |
| `currentBuild.currentResult` | Current pipeline result |

---

## 7. Testing Slack Notifications

Both notification conditions were tested successfully.

### Test 1: Successful Pipeline

Jenkins Build #9 completed successfully.

Slack notification:

```text
SUCCESS: Java_pipeline - Build #9
Build URL: [Jenkins Build URL]
```

Result: **PASSED**

### Test 2: Deliberate Pipeline Failure

Used Jenkins Replay to temporarily insert this command into the Build stage:

```groovy
error('Testing Slack failure notification')
```

This deliberately stops the pipeline and marks the build as failed.

Slack notification:

```text
FAILED: Java_pipeline - Build #10
Build URL: [Jenkins Build URL]
```

Result: **PASSED**

The Replay modification was temporary and was not committed to GitHub.

---

## 8. Jenkins Replay

Replay allows us to execute a modified version of a previous pipeline without modifying the Jenkinsfile in the GitHub repository.

Steps:

1. Open the Jenkins job.
2. Select a previous build.
3. Click Replay.
4. Modify the pipeline script.
5. Click Run.
6. Inspect the build result.

**Use case:** Testing pipeline changes and failure scenarios without committing temporary changes to the main branch.

---

## 9. Day 7 Practical Results

| Task | Status |
|---|---|
| Jenkins pipeline parameters | Completed |
| Conditional Test stage | Completed |
| Maven build goal selection | Completed |
| JUnit test reporting | Completed |
| Pipeline post conditions | Completed |
| Slack workspace and channel setup | Completed |
| Jenkins Slack plugin integration | Completed |
| Secure Slack credential configuration | Completed |
| Slack success notification | Tested successfully |
| Slack failure notification | Tested successfully |
| Jenkins Replay | Tested successfully |

---

# 10. Hands-On Revision Drill

Try these exercises without immediately checking the answers.

### Drill 1 – Pipeline Parameters

1. What is the purpose of a Boolean parameter?
2. What is the difference between a Boolean parameter and a Choice parameter?
3. How do you access a parameter inside a Jenkinsfile?

### Drill 2 – Conditional Execution

Suppose:

```groovy
RUN_TESTS = false
```

Questions:

1. Will the Test stage execute?
2. Which Jenkins directive controls conditional execution?
3. Why would we want to skip tests in some pipeline runs?

### Drill 3 – Post Conditions

Explain the difference between:

```groovy
always
success
failure
```

Which condition would you use to send a success notification? Which one would you use for a failure notification?

### Drill 4 – Slack Integration

Answer these questions:

1. Why do we use Jenkins Credentials instead of hardcoding a Slack token?
2. What does `tokenCredentialId: 'slack-token'` do?
3. Why must the Slack bot be invited into the channel?
4. What does `botUser: true` configure?
5. What is the purpose of the `color` parameter?

### Drill 5 – Environment Variables

Identify the purpose of:

```groovy
env.JOB_NAME
env.BUILD_NUMBER
env.BUILD_URL
```

### Drill 6 – Failure Simulation

What happens when Jenkins executes:

```groovy
error('Testing Slack failure notification')
```

Where can you temporarily insert this command to test the pipeline failure notification?

### Drill 7 – Real-World Scenario

Imagine your Maven application builds successfully but its tests fail.

Answer:

1. Should Jenkins mark the pipeline as successful or failed?
2. Which `post` condition should send the failure notification?
3. How can Jenkins display the test results?
4. How would you provide the failed build URL in Slack?

---

## 11. Important Takeaways

- Jenkins parameters make pipelines configurable.
- The `when` directive controls stage execution.
- `post` conditions automate actions based on build results.
- JUnit helps publish and visualize test reports.
- Slack integration provides real-time pipeline notifications.
- Jenkins Credentials protect sensitive information.
- Jenkins Replay is useful for safely testing pipeline modifications.

##Slack Image
<img width="1391" height="575" alt="ChatGPT Image Nov 13, 2026, 11_46_35 PM" src="https://github.com/user-attachments/assets/0ae0b069-de19-46a0-830b-910d0a8a77c3" />


**Day 7 Status: COMPLETED**

**Project:** Java_pipeline  
**Repository:** `NikhilBCA2022/jenkins-maven-demo`  
**Focus:** Jenkins Advanced Pipeline + Slack Notifications
