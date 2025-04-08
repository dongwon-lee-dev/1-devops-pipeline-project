# Infrastructure Deployment - EC2 Version

[To Home](README.md)

---

## Table of Contents
- [1) Prepare AWS Environment](#1-prepare-aws-environment)
  - [1.1. Create VPC](#1-create-vpc)
  - [1.2. Create Security Groups](#2-create-security-groups)
  - [1.3. Create EC2 Instances on Public Subnet](#3-create-ec2-instances-on-public-subnet)
- [2) Install Applications](#2-install-applications)
  - [2.1. SonarQube (Port 9000): Install on SonarQube Instance](#21-sonarqube-port-9000-install-on-sonarqube-instance)
  - [2.2. Nexus (Port 8081): Install on Nexus Instance](#22-nexus-port-8081-install-on-nexus-instance)
  - [2.3. Kubernetes Master: Install on Kubernetes Master Instance](#23-kubernetes-master-install-on-kubernetes-master-instance)
  - [2.4. Kubernetes Worker (Create & Join): Install on Kubernetes Worker Instance](#24-kubernetes-worker-create--join-install-on-kubernetes-worker-instance)
  - [2.5. Jenkins (Port 8080): Install on Jenkins Instance](#25-jenkins-port-8080-install-on-jenkins-instance)
  - [2.6. Prometheus (Port 9090): Install on Monitoring Instance](#26-prometheus-port-9090-install-on-monitoring-instance)
  - [2.7. Prometheus - Blackbox Exporter (Port 9115): Install on Monitoring Instance](#27-prometheus--blackbox-exporter-port-9115-install-on-monitoring-instance)
  - [2.8. Grafana (Port 3000): Install on Monitoring Instance](#28-grafana-port-3000-install-on-monitoring-instance)



---

# 1) Prepare AWS Environment
### 1.1. Create VPC
1. VPC Dashboard - Create VPC
2. VPC settings
    - Resources to create: VPC and more
    - NAT gateways: In 1 AZ
    - VPC endpoints: None
    - Others are default

### 1.2. Create Security Groups 
| Type | Protocol | Port range | Source |
|---------|----------|----------|----------|
| SSH | TCP  |  22|0.0.0.0/0|
|HTTP|TCP|80|0.0.0.0/0|
| HTTPS  | TCP  |   443|0.0.0.0/0|
| SMTP |TCP|   25 |0.0.0.0/0|
| SMTPS|  TCP  | 465|0.0.0.0/0|
| Custom TCP | TCP |   3000-10000 |0.0.0.0/0|
| Custom TCP |  TCP   |  30000-32767|0.0.0.0/0|

### 1.3. Create EC2 Instances on Public Subnet
<table>
  <tr>
    <th colspan="5" style="background-color: lightgray;">EC2 Instance</th>
  </tr>
  <tr>
    <td>t2.medium gp2 20gb</td>
    <td>SonarQube / Nexus / Kubernetes Master / Kubernetes Worker - Total 4 Instances</td>
  </tr>
  <tr>
    <td>t2.large gp2 20gb</td>
    <td>Jenkins / Monitoring (Prometheus & Prometheus - Blackbox Exporter & Grafana) - Total 2 Instances</td>
  </tr>
</table>

---

# 2) Install Applications

### 2.1. SonarQube (Port 9000): Install on SonarQube Instance
```bash
sudo apt update

# Add Docker's official GPG key:
sudo apt install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

sudo chmod 666 /var/run/docker.sock

docker run -d --restart=always --name sonar -p 9000:9000 sonarqube:lts-community
```
| Default ID | Default Password | 
|---------|----------|
| admin | admin  |  

### 2.2. Nexus (Port 8081): Install on Nexus Instance
```bash
sudo apt update

# Add Docker's official GPG key:
sudo apt install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

sudo chmod 666 /var/run/docker.sock

docker run -d --restart=always --name Nexus -p 8081:8081 sonatype/nexus3
```
| Default ID | Default Password | 
|---------|----------|
| admin | [ /nexus-data/admin.password ] |  

```bash
# Find Container ID
docker ps -a

docker exec -it [Container ID] /bin/bash
cat /nexus-data/admin.password
```

*** Wipe the Nexus releases, snapshots repository after each deployment

### 2.3. Kubernetes Master: Install on Kubernetes Master Instance
```bash
sudo su
```
Create and run the kube-install.sh file

```bash
sudo apt update

sudo apt install docker.io -y
sudo chmod 666 /var/run/docker.sock

sudo apt install -y apt-transport-https ca-certificates curl gnupg
sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update

sudo apt install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1
```

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Copy the output command for worker nodes to join the cluster
kubeadm join 172.31.94.179:6443 --token w0gcqq.ipyxo3aa0t180qqa \
        --discovery-token-ca-cert-hash sha256:b81378544d03994815d38b40bd254087307fda9d6774787676102e6880861cb3
```
```bash
export KUBECONFIG=/etc/kubernetes/admin.conf  

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### ⚠️ When getting the connection refused error when doing 'kubectl get node' such as
```bash
E1017 13:08:17.023750    4693 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
```
Do the following.

```bash
/usr/bin/containerd config default > /etc/containerd/config.toml  

vi /etc/containerd/config.toml 
SystemdCgroup = true   # Change from false to true

sudo systemctl restart containerd.service
```
Check if the error is solved.

Next

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml
```

Extra) Install kubeaudit

```bash
wget https://github.com/Shopify/kubeaudit/releases/download/v0.22.1/kubeaudit_0.22.1_linux_amd64.tar.gz
tar -xvzf kubeaudit_0.22.1_linux_amd64.tar.gz
sudo mv kubeaudit /usr/local/bin/
kubeaudit all
```

### 2.4. Kubernetes Worker (Create & Join): Install on Kubernetes Worker Instance
```bash
sudo su
```
Create and run the kube-install.sh file

```bash
sudo apt update

sudo apt install docker.io -y
sudo chmod 666 /var/run/docker.sock

sudo apt install apt-transport-https ca-certificates curl gnupg -y
sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update

sudo apt install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1
```

Join a worker to the master's cluster
```bash
kubeadm join 172.31.84.207:6443 --token lv44b6.meworri45wtozt38 \
        --discovery-token-ca-cert-hash sha256:027c1955cc63554a9769fa986d0073b013f594038ea660a1227ae7456b15493f
```

To get the Master's Token again
```bash
kubeadm token create --print-join-command
```
### 2.5. Jenkins (Port 8080): Install on Jenkins Instance

#!/bin/bash
```bash
sudo apt update -y

# Install OpenJDK 17 JRE Headless
sudo apt install openjdk-17-jre-headless -y

# Download Jenkins GPG key
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

# Add Jenkins repository to package manager sources
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update package manager repositories
sudo apt update

# Install Jenkins
sudo apt install jenkins -y

sudo systemctl start jenkins
sudo systemctl enable jenkins

# Install kubectl - replace arm64, amd64 according to architecture
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

Install Trivy
```bash
sudo apt install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt update
sudo apt install trivy -y
```
---


```bash
sudo apt update
```

### 2.6. Prometheus (Port 9090): Install on Monitoring Instance
```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.53.2/prometheus-2.53.2.linux-amd64.tar.gz
tar -xvf prometheus-2.53.2.linux-amd64.tar.gz
cd prometheus-2.53.2.linux-amd64
./prometheus &
```
```bash
# Raspberry pi
sudo apt install prometheus
sudo ufw allow 9090
```

### 2.7. Prometheus - Blackbox Exporter (Port 9115): Install on Monitoring Instance
Prometheus Blackbox Exporter: Monitors the status of services externally, even in environments where collecting internal metrics is impossible or difficult.
```bash
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz
tar -xvf blackbox_exporter-0.25.0.linux-amd64.tar.gz
cd blackbox_exporter-0.25.0.linux-amd64
./blackbox_exporter &
```

### 2.8. Grafana (Port 3000): Install on Monitoring Instance
```bash
sudo apt-get install -y adduser libfontconfig1 musl
wget https://dl.grafana.com/enterprise/release/grafana-enterprise_11.2.2_amd64.deb
sudo dpkg -i grafana-enterprise_11.2.2_amd64.deb
sudo /bin/systemctl start grafana-server
```
```bash
# Raspberry Pi
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install -y grafana
sudo /bin/systemctl enable grafana-server
sudo /bin/systemctl start grafana-server
sudo ufw allow 3000
```
---

[To Home](README.md)