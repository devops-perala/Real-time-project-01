# Real Time DevOps Project: Deploy to Kubernetes Using Jenkins | End-to-End CI/CD Pipeline

## Project Overview

This project demonstrates a full-scale DevOps pipeline deploying applications to Kubernetes using Jenkins. It covers EC2 provisioning, Jenkins setup, SonarQube integration, Docker build/push, EKS cluster setup, ArgoCD GitOps configuration, and CI/CD automation.

---

## Jenkins Master Setup

### Launch EC2: Jenkins-Master

* **AMI**: Ubuntu (Free Tier)
* **Instance Type**: t2.micro
* **Storage**: 15 GiB
* **Key Pair**: Existing or create new

### Connect & Configure

```bash
cd Downloads
ssh -i [keypair-name].pem ubuntu@[public-IP]
sudo apt update && sudo apt upgrade -y
sudo hostnamectl set-hostname Jenkins-Master
bash
sudo apt install openjdk-17-jre -y
java -version
```

### Install Jenkins

```bash
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

### Enable SSH Password Authentication

```bash
sudo vi /etc/ssh/sshd_config
# Uncomment and set the following:
# PasswordAuthentication yes
# PermitRootLogin yes
sudo service sshd reload
```

Update Jenkins-Master security group to allow port `8080`.

---

## Jenkins Agent Setup

### Launch EC2: Jenkins-Agent

* Same settings as Jenkins-Master

### Connect & Configure

```bash
cd Downloads
ssh -i [keypair-name].pem ubuntu@[public-IP]
sudo apt update && sudo apt upgrade -y
sudo hostnamectl set-hostname Jenkins-Agent
bash
sudo apt install openjdk-17-jre -y
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER
sudo init 6
```

After reboot, reconnect and configure SSH:

```bash
sudo vi /etc/ssh/sshd_config
# Uncomment PasswordAuthentication & PermitRootLogin
dudo service sshd reload
```

### SSH Key Authentication

From **Jenkins-Master**:

```bash
ssh-keygen
cd .ssh
cat id_rsa.pub
```

On **Jenkins-Agent**:

```bash
cd .ssh
sudo vi authorized_keys  # Paste the copied key
```

---

## Jenkins UI Configuration

* Access Jenkins via `http://[public-IP]:8080`
* Get initial password:

  ```bash
  sudo cat /var/lib/jenkins/secrets/initialAdminPassword
  ```
* Install suggested plugins
* Create first admin user

### Add Jenkins Agent Node

* Jenkins Dashboard → Manage Jenkins → Nodes
* Create new node: Jenkins-Agent
* Remote Root Dir: `/home/ubuntu`
* Launch method: SSH → Add credentials (use `id_rsa` from Jenkins-Master)

---

## Tool Integrations

### Maven

* Manage Jenkins → Global Tool Config → Add Maven (Maven3)

### JDK

* Add JDK: Name - Java17, Install from adoptium.net → jdk-17.0.5+8

### GitHub Credentials

* Manage Jenkins → Credentials → Global → Add Username/Token

---

## CI Pipeline

### Create Jenkins CI Job

* Type: Pipeline
* Definition: Pipeline script from SCM
* SCM: Git → Add forked repo URL
* Credentials: GitHub

---

## SonarQube Setup

### Launch EC2: SonarQube

* **Instance Type**: t3.medium
* **Port 9000**: Allow in Security Group

### Install PostgreSQL & Create DB

```bash
sudo apt update
sudo apt-get -y install postgresql postgresql-contrib
sudo systemctl enable postgresql
sudo passwd postgres
su - postgres
createuser sonar
psql
ALTER USER sonar WITH ENCRYPTED password 'sonar';
CREATE DATABASE sonarqube OWNER sonar;
GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonar;
\q
exit
```

### Install Java 17

```bash
wget -O - https://packages.adoptium.net/... | tee /etc/apt/keyrings/adoptium.asc
apt update
apt install temurin-17-jdk
```

### Kernel Tuning

```bash
sudo vi /etc/security/limits.conf
# Add:
sonarqube - nofile 65536
sonarqube - nproc 4097

sudo vi /etc/sysctl.conf
# Add:
vm.max_map_count = 262144
sudo init 6
```

### Install SonarQube

```bash
wget https://binaries.sonarsource.com/...zip
unzip -d /opt
mv /opt/sonarqube-* /opt/sonarqube
...
```

Update `sonar.properties` with DB URL, username, password.

### Create SystemD Service for SonarQube

```bash
sudo vim /etc/systemd/system/sonar.service
# Add unit/service/install sections
sudo systemctl start sonar
sudo systemctl enable sonar
```

Access: `http://[public-IP]:9000` → admin/admin → update password

---

## Integrate SonarQube with Jenkins

* Generate token in SonarQube → Add to Jenkins as Secret Text
* Install SonarQube Jenkins Plugin
* Configure SonarQube server in Jenkins
* Add Sonar Scanner under Tools
* Create webhook in SonarQube to Jenkins

---

## Docker & EKS Setup

### Add Docker Credentials

* Generate DockerHub token → Add in Jenkins credentials

### Bootstrap EKS Server

* Install AWS CLI, `kubectl`, `eksctl`
* Attach IAM Role with `AdministratorAccess`
* Create EKS Cluster:

```bash
eksctl create cluster --name [cluster-name] --region ap-south-1 --node-type t2.small --nodes 3
```

---

## ArgoCD Setup

### Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/.../install.yaml
```

### Install CLI

```bash
curl -sSL -o /usr/local/bin/argocd https://github.com/.../argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

### Expose ArgoCD

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Login: admin / decoded password from secret

### Add Cluster to ArgoCD

```bash
argocd cluster add [cluster-context-name] --name [name-alias]
```

---

## GitOps with ArgoCD

* Fork GitOps repo: `gitops-register-app`
* Connect GitHub Repo to ArgoCD via HTTPS
* Create Application:

  * Name: register-app
  * Path: ./
  * Cluster: your EKS cluster
  * Sync Policy: Auto (Self-heal, Prune)

### Access Application

```bash
kubectl get svc
# Find external DNS and access: http://[dns]:8080/webapp
```

---

## Jenkins CD Job for GitOps

* Create `GitOps-register-app-cd` job
* Add string param: `IMAGE_TAG`
* Trigger builds remotely with token
* Use pipeline from GitOps repo

### Generate API Token

* User → Configure → Add new token: `JENKINS_API_TOKEN`
* Save & add to Jenkins Credentials

---

## Trigger CI/CD

* Poll SCM → Schedule: `* * * * *`
* Push code to GitHub → Pipeline triggers

---

## 🎉 Done!

Congratulations! You've built a full CI/CD pipeline using Jenkins and Kubernetes with GitOps.

---

## Author

**Rakesh Perala**
Follow and star the repo on GitHub!
