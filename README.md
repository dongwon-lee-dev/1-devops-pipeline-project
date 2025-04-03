# Corporate Level DevOps Pipeline Project

![Pipeline](./images/pipeline.jpg)


✨ **Technologies** ✨
| Technology | Description                                                                                                        |
|------------|--------------------------------------------------------------------------------------------------------------------|
| AWS        | Cloud platform by Amazon offering services like computing, storage, and networking.                                |
| Jenkins    | Open-source automation server used to build and run CI/CD pipelines.                                               |
| GitHub     | Cloud-based platform for hosting Git repositories, enabling version control and collaboration.                     |
| Maven      | Build automation and dependency management tool for Java-based projects.                                           |
| Trivy      | Security scanner that detects vulnerabilities in container images, file systems, and code repositories.            |
| SonarQube  | Static code analysis tool that detects bugs, code smells, and security vulnerabilities.                            |
| Nexus      | Repository manager that stores and distributes built artifacts.                                                    |
| Docker     | Platform that packages applications into containers for consistent and portable execution.                         |
| Kubernetes | Container orchestration platform for automating deployment, scaling, and management of containerized applications. |
| Prometheus | Time-series monitoring tool that collects and queries metrics.                                                     |
| Grafana    | Dashboard tool for visualizing data from various sources, often used with Prometheus.                              |


## Table of Contents
- [1) Prepare AWS](#1-prepare-aws)
  - [1. Create VPC](#1-create-vpc)
  - [2. Create Security Group ](#2-create-security-group)
  - [3. Create EC2 Instances](#3-create-ec2-instances)
  - [4. Run script on SonarQube (Port 9000) instance](#4-run-script-on-sonarqube-port-9000-instance)
  - [5. Run script on Nexus (Port 8081) instance](#5-run-script-on-nexus-port-8081-instance)
  - [6. ☸️ Kubernetes Master Configuration](#6-kubernetes-master-configuration)
  - [⚠️ When getting the connection refused error when doing 'kubectl get node' such as](#️-when-getting-the-connection-refused-error-when-doing-kubectl-get-node-such-as)
  - [7. ☸️ Setting up Kubernetes workers (Create & Join)](#7-setting-up-kubernetes-workers-create--join)
  - [8. Run script on Jenkins (Port 8080) instance](#8-run-script-on-jenkins-port-8080-instance)
- [2) Jenkins Pipeline](#2-jenkins-pipeline)
  - [1. Install Jenkins plugins](#1-install-jenkins-plugins)
  - [2. Jenkins System Configuration](#2-jenkins-system-configuration)
  - [3. Setting up Jenkins Credentials](#3-setting-up-jenkins-credentials)
  - [4. Jenkins pipeline](#4-jenkins-pipeline)
- [3) Monitoring](#3-monitoring)
  - [Install Prometheus (Port 9090)](#install-prometheus-port-9090)
  - [Install Prometheus - Blackbox Exporter (Port 9115)](#install-prometheus---blackbox-exporter-port-9115)
  - [Install Grafana (Port 3000)](#install-grafana-port-3000)
  - [System performance metrics](#system-performance-metrics)
  
<br>
<br>

Credit to jaiswaladi246
https://youtu.be/NnkUGzaqqOc?si=-5ugADFn6lgBzpK9


---

# 1) Prepare AWS
### 1. Create VPC
### 2. Create Security Group 
| Type | Protocol | Port range | Source |
|---------|----------|----------|----------|
| SSH | TCP  |  22|0.0.0.0/0|
|HTTP|TCP|80|0.0.0.0/0|
| HTTPS  | TCP  |   443|0.0.0.0/0|
| SMTP |TCP|   25 |0.0.0.0/0|
| SMTPS|  TCP  | 465|0.0.0.0/0|
| Custom TCP | TCP |   3000-10000 |0.0.0.0/0|
| Custom TCP |  TCP   |  30000-32767|0.0.0.0/0|

### 3. Create EC2 Instances
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
    <td>Jenkins / Monitoring(Prometheus, Grafana) - Total 2 Instances</td>
  </tr>
</table>

### 4. Run script on SonarQube (Port 9000) instance
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

### 5. Run script on Nexus (Port 8081) instance
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

### 6. ☸️ Kubernetes Master Configuration
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

### 7. ☸️ Setting up Kubernetes workers (Create & Join)
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
### 8. Run script on Jenkins (Port 8080) instance

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

# 2) Jenkins Pipeline

Pipeline

## 1. Install Jenkins plugins
<table>
  <tr>
    <th colspan="5" style="background-color: lightgray;">Jenkins Plugins</th>
  </tr>
  <tr>
    <td>JDK</td>
    <td>eclipse temurin installer </td>
  </tr>
  <tr>
    <td>Maven</td>
    <td>config file provider, maven integration, pipeline maven integration</td>
  </tr>
  <tr>
    <td>SonarQube</td>
    <td>sonarqube scanner </td>
  </tr>
    <tr>
    <td>Docker</td>
    <td>Docker, Docker Pipeline</td>
  </tr>
    <tr>
    <td>Kubernetes</td>
    <td>kubernetes, kuberntes cli, kubernetes client api, kubernetes credentials</td>
  </tr>
      <tr>
    <td>Prometheus</td>
    <td>Prometheus Metrics Plugin</td>
  </tr>
</table>

## 2. Jenkins System Configuration
Manage Jenkins - Tools - JDK / Maven / SonarQube / Docker
![Sample](./images/tools/jdk.png)
![Sample](./images/tools/maven.png)
![Sample](./images/tools/sonar.png)
![Sample](./images/tools/docker.png)


## 3. Setting up Jenkins Credentials
<table>
  <tr>
    <th colspan="5" style="background-color: lightgray;">Jenkins Credentials</th>
  </tr>
  <tr>
    <td>sonar-cred (Secret text)</td>
    <td>SonarQube - Administration - Configuration - Users - Tokens</td>
  </tr>
  <tr>
    <td>github-cred (Username with password)</td>
    <td>Github username and token</td>
  </tr>
  <tr>
    <td>docker-cred (Username with password)</td>
    <td>Docker hub username and password</td>
  </tr>
  <tr>
    <td>k8-cred (Secret text)</td>
    <td>kubectl describe secret mysecretname -n webapps</td>
  </tr>
    <tr>
    <td>mail-cred (Username with password)</td>
    <td>Gmail App Password</td>
  </tr>
</table>

## 4. Jenkins pipeline

Refer to jenkins.md file

1. Git Pull (GitHub)
2. Compile (Maven)
3. Test (Maven)
4. File System Scan (Trivy)
5. SonarQube Analysis (SonarQube)
    1. **Get SonarQube Token**: SonarQube - Administration - Security - Users - Administrator Token - Generate Tokens
    2. **Create Jenkins SonarQube Credential**: Jenkins Credentials - create sonar-cred (Secret text) with the token
    3. **Set up Jenkins SonarQube System**: Jenkins System - SonarQube installations - sonarqube / server url / sonar-cred
    4. **Register 3rd party tool in pipeline**: environment { }
6. Quality Gate (SonarQube)
    1. **Register Webhook on SonarQube**: SonarQube - Administration - Configuration - Webhooks - Name: jenkins, URL: [Jenkins URL]/sonarqube-webhook/
8. Build (Maven)
9. Publish To Nexus (Nexus)
    1. Nexus maven-releases, maven-snapshots URL -> Boardgame/pom.xml
    2. Jenkins - Managed files - Add a new Config - Global Maven settings.xml / ID: global-settings - <server> - maven-releases / maven-snapshots
![Sample](./images/managed-files/global-settings.png)
10. Build & Tag Docker Image (Docker)
11. Docker Image Scan (Docker)
12. Push Docker Image (Docker)
13. Deploy To Kubernetes (Kubernetes)
    - Create Service Account, Role, Binding in Kubernetes Cluster
    - Create Secret in Kubernetes Cluster webapps namespace (kubectl apply -f secret.yaml -n webapps)
```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
```
```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
        - ""
        - apps
        - autoscaling
        - batch
        - extensions
        - policy
        - rbac.authorization.k8s.io
    resources:
      - pods
      - secrets
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```
```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role 
subjects:
- namespace: webapps 
  kind: ServiceAccount
  name: jenkins 
```
```bash
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: jenkins
```
```bash
kubectl apply -n webapps -f secret.yaml
```
```bash
kubectl describe secret mysecretname -n webapps
```
14. Verify the Deployment (Kubernetes)
15. Send Email (Gmail)
    1. **Create Gmail App Password**: Manage your Google Account - Security - 2-Step Verification - App passwords
    2. **Jenkins SMTP server configuration**: Manage Jenkins - System - E-mail Notification & Extended E-mail notification
![Sample](./images/system/email-notification.png)
![Sample](./images/system/extended-email-notification.png)


---

# 3) Monitoring
```bash
sudo apt update
```

### Install Prometheus (Port 9090)
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

### Install Prometheus - Blackbox Exporter (Port 9115)
Prometheus Blackbox Exporter: Monitors the status of services externally, even in environments where collecting internal metrics is impossible or difficult.
```bash
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz
tar -xvf blackbox_exporter-0.25.0.linux-amd64.tar.gz
cd blackbox_exporter-0.25.0.linux-amd64
./blackbox_exporter &
```

### Install Grafana (Port 3000)
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

1. Add to prometheus-2.53.2.linux-amd64/prometheus.yml
In case of Raspberry Pi, edit /etc/prometheus/prometheus.yml
```yaml
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]  # Look for a HTTP 200 response.
    static_configs:
      - targets:
        - http://prometheus.io    # Target to probe with http.
        - http://[Kubernetes worker node ip address]:30898
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: [Monitoring Instance ip address]:9115  # The blackbox exporter's real hostname:port.
```

2. Restart Promethius
```bash
pgrep prometheus     
kill [pid]
./prometheus &
```
```bash
# Raspberry Pi
sudo systemctl restart prometheus
```

3. Create Grafana Dashboard admin admin
1. Home > Connections > Data sources > prometheus - Connections - Data sources - Connection Prometheus server URL: http://[ip address]:9090  
*** Change whenever the monitor instance ip address changes
2. Click on the top right + Import Dashboard 7587 Load, signcl-prometheus: prometheus - Import

## System performance metrics 
1.Jenkins plugin: prometheus metrics

2. Jenkins server: install Node Exporter (9100)
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar -xvf node_exporter-1.8.2.linux-amd64.tar.gz
cd node_exporter-1.8.2.linux-amd64.tar.gz
./node_exporter &
``` 
```bash
# Raspberry pi
sudo apt install prometheus-node-exporter
sudo ufw allow 9100
```

3. Jenkins - System - Prometheus configuration default

4. Monitoring server - prometheus.yml add
```yaml
scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['[jenkins server ip]:9100']
  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['[jenkins server ip]:8080']
```

5. Restart Promethius
```bash
pgrep prometheus     
kill [pid]
./prometheus &
```

6. Create a Grafana dashboard: Click on the top right + Import Dashboard 1860 & 9964 Load, signcl-prometheus: prometheus - Import

