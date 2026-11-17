# Jenkins Controller and SSH Agent on AWS

Day 9 of my DevOps/MLOps learning plan. I set up a Jenkins controller with a separate Linux build agent connected over SSH, ran a pipeline on the agent, and practised diagnosing common agent failures.

## Architecture

```
Controller (Jenkins, EC2)  ---- SSH, port 22, private IP ---->  Agent (EC2)
  schedules jobs                                               runs the builds
  stores credentials                                           user: jenkins
                                                               label: linux-agent
```

- **Controller:** Ubuntu EC2 instance running Jenkins, Java 21.
- **Agent:** Ubuntu EC2 instance in the same VPC, Java 21, with a dedicated `jenkins` user.
- **Connection:** the controller logs in to the agent over SSH using a key pair. The agent's security group allows port 22 only from the controller and from my own IP, not from `0.0.0.0/0`.

## Setup steps

1. **Check the network.** From the controller, `nc -zv -w 5 <AGENT_PRIVATE_IP> 22` must report `succeeded`. Use the agent's **private** IP, because a public IP made the connection hang when the security group allowed only the controller's private IP.
2. **Prepare the agent.**
   ```bash
   sudo apt update && sudo apt install -y openjdk-21-jre-headless
   sudo adduser --disabled-password --gecos "" jenkins
   sudo mkdir -p /home/jenkins/agent
   sudo chown jenkins:jenkins /home/jenkins/agent
   ```
3. **Create an SSH key pair** on the controller:
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/jenkins_agent -N "" -C "jenkins-agent"
   ```
4. **Install the public key on the agent** for the `jenkins` user:
   ```bash
   sudo -u jenkins mkdir -p /home/jenkins/.ssh
   echo "<PUBLIC_KEY>" | sudo -u jenkins tee /home/jenkins/.ssh/authorized_keys
   sudo chmod 700 /home/jenkins/.ssh
   sudo chmod 600 /home/jenkins/.ssh/authorized_keys
   ```
5. **Test the login** from the controller before touching Jenkins:
   ```bash
   ssh -i ~/.ssh/jenkins_agent jenkins@<AGENT_PRIVATE_IP> "hostname; whoami; java -version"
   ```
6. **Add the private key to Jenkins** under Manage Jenkins > Credentials (Kind: SSH Username with private key, username `jenkins`).
7. **Create the node** under Manage Jenkins > Nodes > New Node:
   - Name `linux-agent-1`, Permanent Agent
   - Remote root directory `/home/jenkins/agent`
   - Labels `linux-agent`
   - Usage: only build jobs with label expressions matching this node
   - Launch method: Launch agents via SSH, host = agent private IP, credential from step 6
   - Host key verification: manually trusted
8. **Check the log** for `Agent successfully connected and online`.

## Test pipeline

```groovy
pipeline {
  agent { label 'linux-agent' }
  stages {
    stage('Where am I') {
      steps {
        sh 'hostname'
        sh 'whoami'
        sh 'pwd'
        sh 'java -version'
      }
    }
  }
}
```

Result: the console showed `Running on linux-agent-1`, the hostname was the agent's, `whoami` printed `jenkins`, and the build finished with SUCCESS.

## Troubleshooting log

| Symptom | Cause | How I diagnosed it | Fix |
|---|---|---|---|
| Agent offline, `Authentication failed` / `Permission denied (publickey)` | `authorized_keys` had loose permissions (mode 666) or was owned by root | Manual `ssh -i` from the controller failed; `/var/log/auth.log` showed bad ownership or modes | `chown jenkins:jenkins` and `chmod 600` on the file, `chmod 700` on `.ssh` |
| Agent offline, SSH login works but the agent doesn't start | Java not installed on the agent | Manual SSH worked, but `java -version` returned `command not found` | `apt install openjdk-21-jre-headless`, then relaunch the agent |
| Agent offline, connection times out with no error | Firewall rule dropped SSH from the controller | `nc -zv -w 5 <AGENT_IP> 22` timed out; `iptables -L INPUT -n` showed a DROP rule | Deleted the rule with `iptables -D`; checked the security group |
| Build never starts and waits in the queue | The job's label did not match any online node | Compared the job's `agent { label }` with the node's Labels field | Corrected the label |

Rule of thumb:
- A **timeout** means the network or firewall.
- A **permission error** means keys, ownership or file modes.
- **Command not found** means software is missing on the agent.
- A **build that waits** means the label doesn't match a node.

## Jenkins vs GitHub Actions

| Jenkins | GitHub Actions |
|---|---|
| Controller | GitHub's servers |
| Agent | Runner |
| Node label (`agent { label }`) | `runs-on` label |
| Waiting for next available executor | Job stuck in Queued |

## Security notes

- The private key is stored only in the Jenkins credential store, and I deleted the copy from the agent.
- Port 22 on the agent is restricted to the controller and my own IP.
- Builds run as the dedicated `jenkins` user, not as an admin account.
- I stopped both instances after finishing to avoid charges.

## What I learned

- A controller schedules work and an agent runs it, and labels connect the two.
- SSH permissions matter: a correct key is ignored if the file ownership or modes are too open.
- The type of failure (timeout, permission error, missing command, waiting build) points to which layer to check first.
