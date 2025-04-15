# ECS version Monitoring

[To Home](README.md)

---

## Table of Contents

- [Configure Prometheus](#configure-prometheus)  
- [Configure Grafana](#configure-grafana)   

---

## Configure Prometheus

1. Configure application to export metrics (Spring Boot app)
- **pom.xml**
```
    <dependency>
      <groupId>io.micrometer</groupId>
      <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
```
- **src/main/resources/application.properties**
```
management.endpoints.web.exposure.include=prometheus
management.metrics.export.prometheus.enabled=true
```

2. Deploy Application ECS Service with **Service Connect** enabled
![prometheus](./images/prometheus/1.png)

3. Check Application ECS Service's **Client alias DNS:port**
![prometheus](./images/prometheus/2.png)

4. Create a custom Prometheus image
- **prometheus.yml**: Input **Client aliass DNS:port**
```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'spring-app'
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: [<Client alais DNS:port>]
```
- **Dockerfile**
```
FROM bitnami/prometheus
COPY prometheus.yml /etc/prometheus/prometheus.yml
```
Configuration and networking

5. Deploy Prometheus ECS Service with **Service Connect** enabled
- Use the same namespace as the application's namespace.

![prometheus](./images/prometheus/3.png)

6. Check Prometheus  

![prometheus](./images/prometheus/4.png)

---

## Configure Grafana
1. Deploy Grafana ECS Service with **Service Connect** enabled
- Use the same namespace as the prometheus' namespace.

![prometheus](./images/prometheus/5.png)

2. Initialize Grafana 
  - default username: admin  
  - default password: admin

3. Add Prometheus Data Source
- Prometheus source: Input Prometheus ECS Service's **Service connect service**

![prometheus](./images/prometheus/6.png)
![prometheus](./images/prometheus/7.png)
![prometheus](./images/prometheus/8.png)
![prometheus](./images/prometheus/9.png)
![prometheus](./images/prometheus/10.png)

4. Create a Dashboard
- Dashboard ID: 4701

![prometheus](./images/prometheus/11.png)
![prometheus](./images/prometheus/12.png)
![prometheus](./images/prometheus/13.png)



---

[To Home](README.md)
