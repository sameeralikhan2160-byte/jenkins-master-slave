# Jenkins Controller-Agent (Distributed Build) Setup on AWS EC2

A hands-on project where I set up a distributed Jenkins build environment from scratch on AWS EC2 — provisioning two Ubuntu instances, installing Jenkins on the controller, configuring a dedicated build agent over SSH, and running a Freestyle job remotely on the agent.

This repo documents the full journey: infrastructure setup, security group configuration, Jenkins installation, SSH-based controller-agent connection, and a working end-to-end build.

> **Note on naming:** Jenkins' official terminology for this is "controller" and "agent" (the older "master/slave" naming was retired from Jenkins docs and UI a while back). I provisioned the EC2 instances before I'd fully internalized that, so the instance name tags and a couple of screenshots below still show `jenkins-master` / `jenkins-slave` and a node named `myslave`. I've kept those as-is since they're what the screenshots actually show, but everywhere else in this README I'm using the current terms.

---

## 🏗️ Architecture

![Jenkins controller-agent architecture](images/00-architecture-diagram.svg)

- **Controller** hosts the Jenkins web UI, job scheduling, and plugin management.
- **Agent** is a separate EC2 instance that only executes the build steps, connected to the controller via **SSH**.
- Both instances sit in the same VPC for low-latency private communication (`172.31.x.x` private IPs).

---

## 🔐 Security Group Configuration (`jenkins-sec`)

| Type       | Protocol | Port | Purpose                            |
|------------|----------|------|-------------------------------------|
| HTTP       | TCP      | 80   | Standard web access                 |
| SSH        | TCP      | 22   | Remote login + controller↔agent comm |
| Custom TCP | TCP      | 8080 | Jenkins Web UI                      |

![Security group inbound rules](images/01-security-group-jenkins-sec.png)

Both EC2 instances (tagged `jenkins-master` and `jenkins-slave` — see naming note above) were launched as `m7i-flex.large` in `ap-south-1` (Mumbai) and share this security group.

---

## ⚙️ Setup Steps

### 1. Launch EC2 Instances
Two Ubuntu 26.04 LTS instances were launched inside the same VPC, both attached to the `jenkins-sec` security group. One would become the controller, the other the agent.

### 2. Install Java on the Controller
Jenkins needs Java to run, so OpenJDK 21 went in first:
```bash
sudo apt install openjdk-21-jdk -y
```

### 3. Install Jenkins on the Controller
Added the official Jenkins Debian repository and installed it:
```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins -y
```

Checked that the service came up clean:
```bash
sudo systemctl status jenkins
```
`active (running)` — Jenkins was up on port `8080`.

![Jenkins service active and running](images/04-jenkins-service-active.png)

![Both EC2 instances running](images/05-ec2-instances-master-slave.png)

### 4. Initial Jenkins Web Setup
- Accessed Jenkins at `http://<controller-public-ip>:8080`
- Pulled the initial admin password from `/var/lib/jenkins/secrets/initialAdminPassword`
- Created the first admin user through the setup wizard

### 5. Prepare the Agent Node
On the agent instance, I created a dedicated **`jenkins`** system user — this is the account the controller SSHes into to run builds:
```bash
sudo useradd -m -d /var/lib/jenkins -s /bin/bash jenkins
sudo passwd jenkins
sudo nano /etc/sudoers
```

### 6. Generate SSH Keys on the Controller
Switched to the `jenkins` user on the controller and generated a key pair to authenticate with the agent:
```bash
ssh-keygen
```
This created `id_ed25519` (private) and `id_ed25519.pub` (public) under `/var/lib/jenkins/.ssh/`.

![Generating the SSH key pair](images/09-ssh-keygen-on-master.png)

### 7. Connect Controller to Agent
Copied the public key over to the agent's `authorized_keys`, then confirmed connectivity with a manual SSH login:
```bash
ssh jenkins@172.31.10.207
```
The first-time connection prompted for the host fingerprint — accepting it confirmed SSH trust was established between the two boxes.

![SSH connection from controller to agent established](images/11-ssh-master-to-slave-success.png)

### 8. Add SSH Credentials in Jenkins
Under **Manage Jenkins → Credentials**, added:
- **Kind:** SSH Username with private key
- **ID:** `myslavecred`
- **Username:** `jenkins`
- **Private Key:** pasted directly from the controller's `id_ed25519`

![Adding SSH credentials in Jenkins](images/12-jenkins-add-ssh-credential.png)

### 9. Configure the Agent Node in Jenkins
Registered the agent as a new **Node** (named `myslave` — see naming note up top) under **Manage Jenkins → Nodes**, using the SSH credential above to connect. Once online, the dashboard showed both executors:

```
Build Executor Status
 ├── Built-In Node   0/2
 └── myslave         0/1
```

### 10. Run a Test Job on the Agent
Created a Freestyle project (`myfirstjob`) and restricted it to run on the `myslave` node. It built successfully:

```
Building remotely on myslave in workspace /var/lib/jenkins/workspace/myfirstjob
Running on: ip-172-31-10-207
jenkins
/var/lib/jenkins/workspace/myfirstjob
Finished: SUCCESS
```

![Console output showing the build ran on the agent](images/15-console-output-build-on-slave.png)

The console output confirms it — the job ran on the agent, not the controller, so the distributed build setup actually works end-to-end and isn't just configured on paper.

---

## 🧠 Key Concepts Demonstrated

- Provisioning and securing EC2 infrastructure for CI/CD
- Installing and configuring Jenkins from the official package repository (not a pre-built AMI)
- Setting up passwordless SSH authentication between two Linux servers
- Registering and managing a Jenkins agent node
- Using Jenkins' credential store for SSH-based node connections
- Verifying distributed build execution through console output rather than assuming it worked

---

## 📌 Notes / Gotchas I Ran Into

- `useradd` initially rejected `/jenkins` as an invalid username — turned out I'd left a leading slash in by mistake.
- The `jenkins` system user needed its home directory set to `/var/lib/jenkins` specifically, to match Jenkins' default agent root — otherwise SSH keys and workspaces don't resolve correctly.
- Had to explicitly accept the SSH host key fingerprint the first time the controller connected to the agent; skip that and the Jenkins node just refuses to come online with no obvious error pointing at why.

---

## 🚀 Possible Next Steps

- Automate this with **Terraform** for EC2 provisioning, plus a shell/Ansible bootstrap script for the Java + Jenkins install, instead of doing it all through the console
- Replace the Freestyle job with a **Jenkinsfile** (Pipeline as Code)
- Add Docker on the agent to run containerized build steps
- Add a second agent to actually demonstrate load distribution across multiple nodes, not just a single one

---

## 🛠️ Tech Stack

`AWS EC2` · `Ubuntu 26.04 LTS` · `Jenkins 2.555.3` · `OpenJDK 21` · `SSH` · `Bash`
