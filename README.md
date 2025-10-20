![Docker](https://img.shields.io/badge/Container-Docker-blue?logo=docker&logoColor=white)
![Java](https://img.shields.io/badge/Language-JS-yellow?logo=JS)
![License](https://img.shields.io/badge/License-MIT-green)
![Linux](https://img.shields.io/badge/OS-Linux-lightgrey?logo=linux)
![Status](https://img.shields.io/badge/Project-End%20to%20End%20CICD-orange)


<div align="center">
  
 [![Typing SVG](https://readme-typing-svg.demolab.com?font=Fira+Code&size=25&pause=1000&color=00BFFF&center=true&vCenter=true&width=600&lines=welcome+to+my+learning+base!;Always+learning+new+Concepts!;Always+learning+new+Tools!)](https://git.io/typing-svg)

</div>

# Jenkins CI/CD Pipeline with Ansible, Docker & AWS EKS

This project demonstrates a **complete end to end CI/CD pipeline** using **Jenkins**, **Ansible**, **Docker**, and **AWS EKS**.  
It covers every stage  from static code checks and Docker image builds, to container deployment on EKS.

---

## Architecture Overview

**Tools & Technologies:**
- Jenkins (CI/CD Orchestration)
- Ansible (Configuration Management)
- Docker (Containerization)
- AWS EC2 (Hosting Jenkins, Ansible, Docker)
- AWS EKS (Kubernetes Cluster for Production)
- GitHub (Source Code Management)

**Pipeline Summary:**
1. Jenkins pulls source code from GitHub.  
2. Lint check runs on HTML, CSS, and JS using NodeJS linters.  
3. Ansible builds and pushes the Docker image to DockerHub.  
4. The container is deployed either:
   - On a Docker host (test/staging), or  
   - On an AWS EKS cluster (production).  

---

## Phase 1 – Jenkins Server Installation 

### 1️. Install Java and Jenkins
Jenkins is a Java-based application, so Java must be installed first.

```bash
sudo dnf install java-11-openjdk -y
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo dnf install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
````

### 2️. Install Git

Required for Jenkins to pull code from GitHub:

```bash
sudo dnf install git -y
```

### 3️. Jenkins Plugins

* **GitHub Plugin** – for SCM integration.
* **NodeJS Plugin** – to run frontend linting tasks.

---

## Static Site Lint & Deploy your first Job

**Project Type:** Freestyle
**Job Name:** `Static-Site-Lint-and-Deploy`

1. Add **Source Code Management (Git)** with your repository URL and correct branch.
2. Enable **Poll SCM** to trigger builds automatically on commits.
3. In the **Build** section, choose:

   * Provide NodeJS and npm folder path.
   * Add a build shell step:

```bash
# Install linters
npm install -g htmlhint stylelint eslint stylelint-config-standard

echo "Running HTML lint"
htmlhint .

echo "Running CSS lint"
stylelint "css/custom/*.css" || echo "No custom CSS to lint"

echo "Running JS lint"
eslint . --ext .js || echo "No JS lint errors"
```

---

##  Phase 2 – Integrate Jenkins with Docker Host via Ansible for test env

### Docker Host Setup (EC2)

1. Install Docker:

   ```bash
   sudo dnf install docker -y
   sudo systemctl enable docker
   sudo systemctl start docker
   ```
2. Create `dockeradmin` user and add SSH key for Jenkins.
3. Install **Publish Over SSH Plugin** in Jenkins and configure Docker host credentials.

---

##  Ansible Server Setup

1. Create Ansible server EC2 instance.
2. Add user:

   ```bash
   useradd ansadmin
   passwd ansadmin
   visudo   # give sudo permissions
   ```
3. Edit `/etc/ssh/sshd_config` → allow password auth, reload SSH:

   ```bash
   systemctl restart sshd
   ```
4. Login as `ansadmin` and install Ansible:

   ```bash
   sudo dnf install ansible -y
   ```

---

##  Configure Ansible Inventory & SSH Access

On **Ansible Server**:

```bash
sudo nano /etc/ansible/hosts
```

```ini
[ansible]
172.31.16.99

[dockerhost]
172.31.19.20
```

Generate SSH keys and copy to target:

```bash
ssh-keygen
ssh-copy-id ansadmin@172.31.19.20 #which we have to create for dockerhost
```

---

##  Test Connection

```bash
ansible all -a uptime
```

Expected Output:

```
172.31.19.20 | CHANGED | rc=0 >>
17:52:59 up 3:38, 3 users, load average: 0.00, 0.00, 0.00
```

---

##  Ansible Playbooks

###  Build & Push Docker Image to registry (`buildandpush.yml`)

```yaml
---
- hosts: ansible
  tasks:
    - name: remove old images
      command: docker image prune -a -f
      ignore_errors: yes

    - name: remove extra index
      command: cd /opt/docker ; sudo rm index.htm
      ignore_errors: yes

    - name: build new image
      command: docker build -t app:latest .
      args:
        chdir: /opt/docker

    - name: tag the image
      command: docker tag app:latest aliaceofali/app:latest

    - name: push to DockerHub
      command: docker push aliaceofali/app:latest
```

###  Deploy to Docker Host (`deploy.yml`)  (The test environment)

```yaml
---
- hosts: dockerhost
  tasks:
    - name: stop existing container
      command: docker stop app-prod
      ignore_errors: yes

    - name: remove old container
      command: docker rm -f app-prod
      ignore_errors: yes

    - name: remove old image
      command: docker rmi aliaceofali/app:latest
      ignore_errors: yes

    - name: run new container
      command: docker run -d --name app-prod -p 8082:80 aliaceofali/app:latest
```

---

##  Phase 3 – Deploy to AWS EKS

###  Setup Bootstrap Server

Install tools:

```bash
# AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# kubectl
curl -o kubectl https://amazon-eks.s3.amazonaws.com/latest/bin/linux/amd64/kubectl
chmod +x ./kubectl && sudo mv kubectl /usr/local/bin/

# eksctl
curl -sLO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz"
tar -xzf eksctl_Linux_amd64.tar.gz && sudo mv eksctl /usr/local/bin/
```

###  Create Cluster

```bash
eksctl create cluster \
  --name testproject \
  --region eu-north-1 \
  --node-type t3.medium \
  --nodes-min 2 \
  --nodes-max 2 \
  --zones eu-north-1a,eu-north-1b
```

Update kubeconfig:

```bash
aws eks --region eu-north-1 update-kubeconfig --name testproject
```

---

##  Kubernetes Manifests

### `deployment.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-prod
  labels:
    app: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: prod
  template:
    metadata:
      labels:
        app: prod
    spec:
      containers:
        - name: app-prod
          image: aliaceofali/app:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80
```

### `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-prod
  labels:
    app: prod
spec:
  selector:
    app: prod
  ports:
    - port: 8080
      targetPort: 80
  type: LoadBalancer
```

### `k8splaybook.yml`

```yaml
---
- hosts: k8s
  become: yes
  become_user: ec2-user
  tasks:
    - name: apply deployment
      command: kubectl apply -f /home/ec2-user/deployment.yml
      ignore_errors: yes

    - name: apply service
      command: kubectl apply -f /home/ec2-user/service.yaml
      ignore_errors: yes

    - name: restart deployment
      command: kubectl rollout restart deployment app-prod
```

To delete the cluster:

```bash
eksctl delete cluster --name testproject --region eu-north-1
```

---

##  Dockerfile Example

```Dockerfile
FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY . /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

##  Jenkins Troubleshooting

If Jenkins web UI breaks or slow after restart:

```bash
cd /var/lib/jenkins/
sudo nano jenkins.model.JenkinsLocationConfiguration.xml
```

Ensure:

```xml
<jenkinsUrl>http://ec2-user@ec2-13-61-181-117.eu-north-1.compute.amazonaws.com:8080</jenkinsUrl>
```

---

###  Docker Issues

#### 1️. Docker Socket Permission Denied

If you get:

```
Got permission denied while trying to connect to the Docker daemon socket
```

Fix:

```bash
sudo chmod 777 /var/run/docker.sock
```

You can also give user group access:

```bash
sudo usermod -aG docker ansadmin
newgrp docker
```

#### 2️. Docker Directory Permission

If Docker build or copy fails:

```bash
sudo mkdir -p /opt/docker
sudo chown ansadmin:ansadmin /opt/docker
```

#### 3️. Docker Service Not Starting

```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker
```

---

##  Demo


* **Full Pipeline:**
  ![Full Pipeline Demo](https://media0.giphy.com/media/v1.Y2lkPTc5MGI3NjExZThldnI2ZnFxeXZheGlydjM0OTFqOW92NDl1dzh2cTd6Z3Zrdzc2cyZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/MYNlxNvC6a5iHDgyJz/giphy.gif)

* **Docker Host Deployment:**
  ![Host Deployment Demo](https://media3.giphy.com/media/v1.Y2lkPTc5MGI3NjExOHZvOHNpaXgya2ZjcWlqdHByb2pmaW9vODJ2dzU5a3hia2h2MDEyYyZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/5PyYtdcX1Hay2mnBnx/giphy.gif)

---

##  Summary

This project demonstrates:

* End-to-end CI/CD automation using Jenkins and Ansible.
* Dockerized application build and push to DockerHub.
* Continuous deployment to Docker host or Kubernetes (EKS).
* Infrastructure setup entirely automated with Ansible.


---
## 👨‍💻 Author
<div align="center">
Ali Khaled
🚀 DevOps Student | ☁️ Cloud & Automation Enthusiast

💬 Learning one container at a time!
<dev>

<p align="center"> <a href="https://github.com/AliKhaledElbaqly" target="_blank"><img src="https://img.shields.io/badge/GitHub-@AliKhaled-black?logo=github"></a> <a href="https://www.linkedin.com/in/alialbaqly/" target="_blank"><img src="https://img.shields.io/badge/LinkedIn-Ali%20Khaled-blue?logo=linkedin"></a> <a href="https://alikhaled.info/" target="_blank"><img src="https://img.shields.io/badge/About_Me-Click_Here-blueviolet?logo=aboutdotme"></a> </p>
<p align="center"> <b>🌍 DevOps Learning Journey — Home Lab 2025</b> </p> 




<h1 align="center">
  Thanks for visiting my little repo , Take care of yourself 
  <img src="https://raw.githubusercontent.com/aemmadi/aemmadi/master/wave.gif" width="30px">
</h1>

<div align="center">
  
[![Typing SVG](https://readme-typing-svg.demolab.com?font=Fira+Code&size=25&pause=1000&color=00BFFF&center=true&vCenter=true&width=600&lines=FREE+PALESTINE)](https://git.io/typing-svg)

</div>

---
