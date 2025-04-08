# GitHub Actions version Pipeline
### ⭐ GitHub Actions-provided actions
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

### 1. Create a workflow yml file in /.github/workflows folder
For example, /.github/workflows/maven.yml

### Configure execution condition
```
on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
```
### 2. Configure execution environment
```
jobs:
  docker-build:
    runs-on: ubuntu-latest
```

### 3. Run SonarQube scan
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

### 4. Run Trivy Security Scan
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

### 5. Push Docker Image to AWS ECR
⭐ Use awslabs/amazon-ecr-credential-helper to securely use AWS Credentials.
[ECR Credential Helper](https://github.com/awslabs/amazon-ecr-credential-helper)

1. Create GitHub Repository Secrets
    - AWS_ACCESS_KEY_ID
    - AWS_SECRET_ACCESS_KEY
```
    - name: Install Docker Credential Helper
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