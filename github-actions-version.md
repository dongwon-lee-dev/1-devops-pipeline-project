# GitHub Actions version Pipeline
[To Home](README.md)

---

## Table of Contents

- [GitHub Actions-provided actions](#github-actions-provided-actions)
- [1. Create a workflow yml file in /.github/workflows folder](#1-create-a-workflow-yml-file-in-githubworkflows-folder)
- [2. Configure execution condition](#2-configure-execution-condition)
- [3. Configure execution environment](#3-configure-execution-environment)
- [4. Run SonarQube scan](#4-run-sonarqube-scan)
- [5. Push to Nexus](#5-push-to-nexus)
- [6. Run Trivy Security Scan](#6-run-trivy-security-scan)
- [7. Push Docker Image to AWS ECR](#7-push-docker-image-to-aws-ecr)


---

### GitHub Actions-provided actions
| Action Name                     | Description                                       |
|---------------------------------|---------------------------------------------------|
| `actions/checkout`              | Checks out your repository for use in the workflow. |
| `actions/upload-artifact`       | Uploads artifacts generated in your workflow.    |
| `actions/download-artifact`     | Downloads artifacts stored from previous runs.   |
| `actions/setup-node`            | Sets up a Node.js environment.                    |
| `actions/setup-python`          | Sets up a Python environment.                     |
| `actions/setup-java`            | Sets up a Java environment.                       |
| `actions/cache`                 | Saves and restores dependency caches to speed up builds. |
| `actions/create-release`        | Creates a GitHub release.                         |
| `actions/upload-release-asset`  | Uploads assets to the created release.            |

---

### 1. Create a workflow yml file in /.github/workflows folder
For example, /.github/workflows/maven.yml

### 2. Configure execution condition
```
on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
```
### 3. Configure execution environment
```
jobs:
  docker-build:
    runs-on: ubuntu-latest
```

### 4. Run SonarQube scan
⭐ Use SonarSource/sonarqube-quality-gate-action
[SonarQube Quality Gate Action](https://github.com/SonarSource/sonarqube-quality-gate-action)

1. Create GitHub Repository Secrets
    - SONAR_TOKEN
    - SONAR_HOST_URL
2. Modify '-Dsonar.projectKey=[your own project key]' according to your project.

```
    - name: Cache Sonar packages
      uses: actions/cache@v3
      with:
        path: ~/.sonar/cache
        key: ${{ runner.os }}-sonar

    - name: Build (compile)
      run: mvn clean compile

    - name: SonarQube Scan
      uses: sonarsource/sonarqube-scan-action@master
      with:
        args: >
          -Dsonar.projectKey=[your own project key]
          -Dsonar.sources=src
          -Dsonar.java.binaries=target/classes
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

          
    - name: SonarQube Quality Gate check
      id: sonarqube-quality-gate-check
      uses: sonarsource/sonarqube-quality-gate-action@master
      with:
        pollingTimeoutSec: 600
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

    - name: "Example show SonarQube Quality Gate Status value"
      run: echo "The Quality Gate status is ${{ steps.sonarqube-quality-gate-check.outputs.quality-gate-status }}"
```

### 5. Push to Nexus
1. Create GitHub Repository Secrets
    - MAVEN_USERNAME
    - MAVEN_PASSWORD
```
    - name: Create Maven settings.xml
      run: |
        cat << EOF > ~/.m2/settings.xml
        <settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
                  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
          <servers>
            <server>
              <id>healthcheck-service-release</id>
              <username>${{ secrets.MAVEN_USERNAME }}</username>
              <password>${{ secrets.MAVEN_PASSWORD }}</password>
            </server>
            <server>
              <id>healthcheck-service-snapshot</id>
              <username>${{ secrets.MAVEN_USERNAME }}</username>
              <password>${{ secrets.MAVEN_PASSWORD }}</password>
            </server>
          </servers>
        </settings>
        EOF

      
    - name: Build and deploy to Nexus
      run: mvn --batch-mode deploy
      env:
        MAVEN_USERNAME: ${{ secrets.MAVEN_USERNAME }}
        MAVEN_PASSWORD: ${{ secrets.MAVEN_PASSWORD }}
```

### 6. Run Trivy Security Scan
```
    - name: Build Docker image
      run: docker build -t [Image Name] .
    - name: Install Trivy
      run: |
        sudo apt-get update
        sudo apt-get install -y apt-transport-https curl gnupg lsb-release
        curl -fsSL https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
        echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/trivy.list
        sudo apt-get update
        sudo apt-get install -y trivy
        
    - name: Scan the Docker image with Trivy
      run: |
        trivy image --format json --output trivy-report.json [Image Name]:latest
        
    - name: Upload Trivy scan results
      uses: actions/upload-artifact@v4
      with:
        name: trivy-report
        path: trivy-report.json
```

### 7. Push Docker Image to AWS ECR
⭐ Use awslabs/amazon-ecr-credential-helper to securely use AWS Credentials. ECR Credential Helper

1. Create GitHub Repository Secrets.
    - AWS_ACCESS_KEY_ID
    - AWS_SECRET_ACCESS_KEY
2. Push the docker image to ECR.
```
    - name: Install ECR Credential Helper
      run: |
        sudo apt-get install amazon-ecr-credential-helper
        mkdir -p ~/.docker
        echo '{"credsStore": "ecr-login"}' > ~/.docker/config.json

    - name: Tag Docker image for ECR
      run: |
        docker tag [Image Name]:latest [ECR Repository URL]

    - name: Push Docker image to AWS ECR
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      run: |
        docker push [ECR Repository URL]
```

### 8. Redeploy ECS Service
1. Create a task definition that pulls image from ECR.
2. Create a ECS service with the created task definition
3. Force redeploy the ECS service to make it use the latest image.
```
aws ecs update-service --region us-east-1 --cluster app-cluster --service [Service Name] --force-new-deployment
```