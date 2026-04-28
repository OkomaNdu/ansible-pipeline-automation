# Ansible Pipeline Automation

This project contains solutions to the Ansible module exercises of the DevOps bootcamp.

---

## Exercise 1: Build & Deploy Java Artifact

Builds a Java Gradle application locally and deploys the jar to a remote Ubuntu server. Creates a Linux user on the remote server if it doesn't exist. On re-runs, stops the running app and removes the old jar before deploying the new one.

### Architecture

```mermaid
graph LR
    A[Local Machine\nAnsible Control Node] -->|gradle clean build| B[Java Gradle Project\nbuild/libs/*.jar]
    A -->|SSH: copy jar + start app| C[Remote Ubuntu Server\n159.203.30.6]
    C --> D[Java App\nrunning as user: ndu]
```

### Configuration

**`ansible.cfg`** — disables SSH host key checking, sets default inventory to `hosts`.

**`hosts`** — defines the `web_server` group pointing to the remote server with SSH key and user.

### Prerequisites

On the remote server, install the `acl` package:
```bash
sudo apt-get update -y && sudo apt-get install -y acl
```

Verify Ansible connectivity:
```bash
ansible -i hosts web_server -m ping
```

### Execution

```bash
ansible-playbook -i hosts 1-build-and-deploy.yaml \
  --extra-vars "linux_user=ndu \
  project_dir=/home/ndu/DevOps-Project/Build-Tools/Java-Build-Tools-Project \
  jar_name=build-tools-exercises-1.0-SNAPSHOT.jar"
```

### What the Playbook Does

1. Creates Linux user on the remote server
2. Installs Java 17 on the remote server
3. Builds the jar locally with `gradle clean build`
4. Stops the running app and removes the old jar (skipped on first run)
5. Copies the new jar, starts the app, and verifies it is running

### Verify Deployment

SSH into the server and check the running process:

```bash
ssh root@159.203.30.6
ps aux | grep java
```

Output:
```
ndu   4348  1.4  6.2 2602804 125308 ?  Sl  03:37  0:09 java -jar build-tools-exercises-1.0-SNAPSHOT.jar &
```

The application is running as user `ndu` on the remote server.

---

## Exercise 2: Push Java Artifact to Nexus

Pushes a locally built jar artifact to a Nexus Repository Manager `maven-snapshots` repository using an HTTP PUT request. The playbook runs entirely on localhost — no remote server required.

### Architecture

```mermaid
graph LR
    A[Local Machine\nAnsible Control Node] -->|HTTP PUT| B[Nexus Repository Manager\nlocalhost:8081\nDocker Container]
    B --> C[maven-snapshots\ncom/my/build-tools-exercises\n1.0-SNAPSHOT]
```

### Prerequisites

Nexus running as a Docker container:
```bash
docker run -d -p 8081:8081 --name nexus sonatype/nexus3
```

Get the initial admin password:
```bash
docker exec nexus cat /nexus-data/admin.password
```

Log in at `http://localhost:8081`, complete the setup wizard, and confirm the `maven-snapshots` repository exists under **Browse**.

### Execution

```bash
ansible-playbook -i hosts 2-push-to-nexus.yaml \
  --extra-vars "nexus_url=http://localhost:8081 \
  nexus_user=admin \
  nexus_password=<your-password> \
  repository_name=maven-snapshots \
  artifact_name=build-tools-exercises \
  artifact_version=1.0-SNAPSHOT \
  jar_file_path=/home/ndu/DevOps-Project/Build-Tools/Java-Build-Tools-Project/build/libs/build-tools-exercises-1.0-SNAPSHOT.jar"
```

### What the Playbook Does

1. Authenticates to Nexus using basic auth
2. Uploads the jar via HTTP PUT to `maven-snapshots` under path `com/my/build-tools-exercises/1.0-SNAPSHOT/`
3. Returns HTTP 201 on success

### Verify Deployment

Browse to **Nexus → Browse → maven-snapshots** and confirm the artifact is present.

![Nexus maven-snapshots](images/nexus-snapshot.png)

---

## Exercise 3: Install Jenkins on EC2 (Amazon Linux)

Provisions an Amazon Linux EC2 instance on AWS and installs Jenkins, Docker, and Node.js on it using two separate playbooks — one for provisioning the server and one for configuring it.

### Architecture

```mermaid
graph LR
    A[Local Machine\nAnsible Control Node] -->|amazon.aws module| B[AWS EC2\nAmazon Linux 2023\nt2.medium]
    A -->|writes public IP| C[hosts-jenkins-server]
    A -->|SSH: install + configure| B
    B --> D[Jenkins :8080]
    B --> E[Docker]
    B --> F[Node.js v8.0.0\nvia NVM]
```

### Prerequisites

- AWS credentials configured: `aws configure`
- Python boto3 installed: `sudo apt install python3-boto3`
- Ansible AWS collection: `ansible-galaxy collection install amazon.aws`
- EC2 key pair created and `.pem` file available locally
- `hosts-jenkins-server` file created: `touch hosts-jenkins-server`
- Port 8080 open in AWS Security Group

### Step 1 — Provision EC2 Instance

```bash
ansible-playbook 3-provision-jenkins-ec2.yaml \
  --extra-vars "ssh_key_path=/home/ndu/Downloads/ansible.pem \
  aws_region=ca-central-1 \
  key_name=ansible \
  subnet_id=subnet-0012ce5d3a6028493 \
  ami_id=ami-0495a76ecf381a767 \
  ssh_user=ec2-user"
```

This will:
1. Query VPC information in the specified region
2. Create an EC2 `t2.medium` instance tagged `jenkins-server` with a public IP
3. Wait 60 seconds for the public IP to be assigned
4. Write the public IP into `hosts-jenkins-server`

### Step 2 — Install and Configure Jenkins

Wait 2-3 minutes for the instance to fully boot, then run:

```bash
ansible-playbook -i hosts-jenkins-server 3-install-jenkins-ec2.yaml \
  --extra-vars "aws_region=ca-central-1"
```

This will:
1. Install Java 21 (Amazon Corretto) and set it as default
2. Add Jenkins repository and install Jenkins
3. Install Docker
4. Install Node.js v8.0.0 via NVM
5. Start Jenkins and print the initial admin password

### What the Playbook Does

| Play | Action |
|------|--------|
| Get server IP | Queries AWS for the `jenkins-server` public IP |
| Prepare server | Installs Java 21, Jenkins, Docker, Node.js |
| Start Jenkins | Starts Jenkins service, verifies port 8080, prints admin password |

### Verify Deployment

Access Jenkins in browser:
```
http://<ec2-public-ip>:8080
```

Get the admin password from playbook output (base64 encoded) and decode it:
```bash
echo "<base64-password>" | base64 --decode
```

**Actual run output:**
```
Jenkins listening on port 8080
Admin password: 84794b58b27c4c7990fa4f38c9366ff4
EC2 Public IP: 16.52.168.187
```

![Jenkins Dashboard](images/jenkins-snapshot.png)
