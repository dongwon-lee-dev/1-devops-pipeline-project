# Corporate Level DevOps Pipeline Project

![Pipeline](./images/pipeline.jpg)

✨ **Technologies** ✨
| Technology | Description |
|------------|--------------------------------------------------------------------------------------------------------------------|
| AWS | Cloud platform by Amazon offering services like computing, storage, and networking. |
| Jenkins | Open-source automation server used to build and run CI/CD pipelines. |
| GitHub | Cloud-based platform for hosting Git repositories, enabling version control and collaboration. |
| Maven | Build automation and dependency management tool for Java-based projects. |
| Trivy | Security scanner that detects vulnerabilities in container images, file systems, and code repositories. |
| SonarQube | Static code analysis tool that detects bugs, code smells, and security vulnerabilities. |
| Nexus | Repository manager that stores and distributes built artifacts. |
| Docker | Platform that packages applications into containers for consistent and portable execution. |
| Kubernetes | Container orchestration platform for automating deployment, scaling, and management of containerized applications. |
| Prometheus | Time-series monitoring tool that collects and queries metrics. |
| Grafana | Dashboard tool for visualizing data from various sources, often used with Prometheus. |

---

## Table of Contents

- [1) Deploy Infrastructure](#1-deploy-infrastructure)
  - [ECS version](#ecs-version)
  - [EC2 version](#ec2-version)
- [2) Configure Jenkins ](#2-configure-jenkins)
  - [1. Install Jenkins plugins](#1-install-jenkins-plugins)
  - [2. Jenkins System Configuration](#2-jenkins-system-configuration)
  - [3. Setting up Jenkins Credentials](#3-setting-up-jenkins-credentials)
- [3) Build Jenkins pipeline](#3-build-jenkins-pipeline)
- [4) Set up Monitoring](#4-set-up-monitoring)
  - [Install Prometheus (Port 9090)](#install-prometheus-port-9090)
  - [Install Prometheus - Blackbox Exporter (Port 9115)](#install-prometheus---blackbox-exporter-port-9115)
  - [Install Grafana (Port 3000)](#install-grafana-port-3000)
  - [System performance metrics](#system-performance-metrics)
- [5) GitHub Actions version](#5-github-actions-version)

<br>
<br>

Credit to jaiswaladi246
https://youtu.be/NnkUGzaqqOc?si=-5ugADFn6lgBzpK9

---

# 1) Deploy Infrastructure

Deploy infrastructure using **ECS version** or **EC2 version**.

## 1. ECS version

[ECS version](https://github.com/dongwon-lee-dev/terraform-devops-pipeline)

![Pipeline](./images/pipeline-ecs-version.jpg)

## 2. EC2 version

[EC2 version](ec2-version.md)

---

# 2) Configure Jenkins

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

## 2. Configure Jenkins Tools

Manage Jenkins - Tools

- JDK installations
- Maven installations
- SonarQube Scanner installations
- Docker installations

![Sample](./images/tools/jdk.png)
![Sample](./images/tools/maven.png)
![Sample](./images/tools/sonar.png)
![Sample](./images/tools/docker.png)

## 3. Configure Jenkins Credentials

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

# 3) Build Jenkins Pipeline

## How to pull from Git Repository
```
steps {
  git branch: 'main', url: '[Repository URL]'
}
```

## How to run AWS Commands

```
steps {
  withCredentials([
      [
        $class: 'AmazonWebServicesCredentialsBinding',
        credentialsId: '088351136602',
        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
      ]
  ]) {
    sh '[AWS Commands]'
  }
}
```

## How to send a Notification Email
1. **Create Recipient Gmail App Password**: Manage your Google Account - Security - 2-Step Verification - App passwords 
2. **Jenkins SMTP server configuration**: Manage Jenkins - System - E-mail Notification & Extended E-mail notification

![Sample](./images/system/email-notification.png)
![Sample](./images/system/extended-email-notification.png)

```
post {
  always {
    script {
      def body = [Email Body]

      emailext (
          subject: [Email Subject],
          body: body,
          to: [Recipient],
          from: 'jenkins@example.com',
          replyTo: 'jenkins@example.com',
          mimeType: 'text/html',
          attachmentsPattern: [File to Attach]
      )
    }
  }
}
```

## Full Pipeline

Refer to jenkins.md file

Steps

1. Git Pull (GitHub)
2. Compile (Maven)
3. Test (Maven)
4. File System Scan (Trivy)
5. SonarQube Analysis (SonarQube)
   1. Create **SonarQube Token**: SonarQube - Administration - Security - Users - Administrator: Tokens - Generate Tokens - Copy the token and save it somewhere
   2. Create Jenkins **SonarQube Credential**: Jenkins Credentials - create sonar-cred (Secret text) with the token
   3. Configure **Jenkins System**: Jenkins System - SonarQube installations - sonarqube / server url / sonar-cred
   4. **Register 3rd party tool in pipeline**: environment { }
6. Quality Gate (SonarQube)
   1. **Register Webhook on SonarQube**: SonarQube - Administration - Configuration - Webhooks - Name: jenkins, URL: [Jenkins URL]/sonarqube-webhook/
7. Build (Maven)
8. Publish To Nexus (Nexus) 1. Nexus maven-releases, maven-snapshots URL -> Boardgame/pom.xml 2. Jenkins - Managed files - Add a new Config - Global Maven settings.xml / ID: global-settings - <server> - maven-releases / maven-snapshots
   ![Sample](./images/managed-files/global-settings.png)
9. Build & Tag Docker Image (Docker)
10. Docker Image Scan (Docker)
11. Push Docker Image (Docker)
12. Deploy To Kubernetes (Kubernetes)
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

---

# 4) Set up Monitoring

## Configure Prometheus
1. Add to prometheus-2.53.2.linux-amd64/prometheus.yml

\* In case of Raspberry Pi, edit /etc/prometheus/prometheus.yml

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

## Configure Grafana

1. Create Grafana Dashboard: admin admin
2. Home > Connections > Data sources > prometheus - Connections - Data sources - Connection Prometheus server URL: http://[ip address]:9090
   - Change whenever the monitor instance ip address changes
3. Click on the top right + Import Dashboard 7587 Load, signcl-prometheus: prometheus - Import

## How to connect Jenkins to Prometheus

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
  - job_name: "node_exporter"
    static_configs:
      - targets: ["[jenkins server ip]:9100"]
  - job_name: "jenkins"
    metrics_path: "/prometheus"
    static_configs:
      - targets: ["[jenkins server ip]:8080"]
```

5. Restart Promethius

```bash
pgrep prometheus
kill [pid]
./prometheus &
```

6. Create a Grafana dashboard: Click on the top right + Import Dashboard 1860 & 9964 Load, signcl-prometheus: prometheus - Import

# 5) GitHub Actions version

[GitHub Actions version](github-actions-version.md)