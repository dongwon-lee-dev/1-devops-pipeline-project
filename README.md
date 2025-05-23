# Corporate Level DevOps Pipeline Project

![Pipeline](./images/pipeline.jpg)

---

## Table of Contents

- [1) Deploy Infrastructure](#1-deploy-infrastructure)
  - [ECS version](#ecs-version)
  - [EC2 version](#ec2-version)
- [2) Deploy Pipeline](#2-deploy-pipeline)
  - [Jenkins version](#jenkins-version)
  - [GitHub Actions version](#github-actions-version)
  - [GitLab CI/CD version](#gitlab-cicd-version)
- [3) Set Up Metrics Monitoring - Prometheus & Grafana](#3-set-up-metrics-monitoring---prometheus--grafana)
  - [ECS Version](#ecs-version)
  - [EC2 Version](#ec2-version)
- [4) Set up Log Monitoring - ELK](#4-set-up-log-monitoring---elk)
  - [ELK EC2 Version](#elk-ec2-version)

---
![Project Introduction](./images/project-introduction.jpg)

---

# 1) Deploy Infrastructure

Deploy infrastructure using **ECS version** or **EC2 version**.

## ECS version

[ECS version](https://github.com/dongwon-lee-dev/terraform-devops-pipeline)

![Pipeline](./images/pipeline-ecs-version.jpg)

## EC2 version

[EC2 version](ec2-version.md)

---

# 2) Deploy Pipeline
Deploy pipeline using **Jenkins version** or **GitHub Actions version**.


## Jenkins version

[Jenkins version](jenkins.md)


## GitHub Actions version

[GitHub Actions version](github-actions-version.md)

## GitLab CI/CD version

[GitLab CI/CD version](gitlab-cicd-version.md)

---

# 3) Set up Metrics Monitoring - Prometheus & Grafana

## ECS Version
[ECS version](ecs-monitoring.md)

## EC2 Version
[EC2 version](ec2-monitoring.md)

--- 

# 4) Set up Log Monitoring - ELK

![ELK](./images/elk.jpg)

ELK (Elasticsearch, Logstash, Kibana)
- Logstash: Collects, processes, and transforms logs before sending them to Elasticsearch.
- Elasticsearch: Stores and indexes log data for fast search and analysis.
- Kibana: Visualizes data stored in Elasticsearch through dashboards and graphs.

## ELK EC2 Version
[EC2 version](elk.md)