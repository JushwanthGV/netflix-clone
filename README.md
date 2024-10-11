
# Deploy Netflix Clone on Cloud using Jenkins - DevSecOps Project!

### Phase 1: Initial Setup and Deployment

**Step 1: Launch EC2 (Ubuntu 22.04):**

- Provision an EC2 instance on AWS with Ubuntu 22.04.
- Connect to the instance using SSH.

**Step 2: Clone the Code:**

- Update all the packages and then clone the code.
- Clone your application's code repository onto the EC2 instance:
    
    ```bash
    git clone https://github.com/N4si/DevSecOps-Project.git
    ```

**Step 3: Install Docker and Run the App Using a Container:**

- Set up Docker on the EC2 instance:
    
    ```bash
    sudo apt-get update
    sudo apt-get install docker.io -y
    sudo usermod -aG docker $USER  # Replace with your system's username, e.g., 'ubuntu'
    newgrp docker
    sudo chmod 777 /var/run/docker.sock
    ```
    
- Build and run your application using Docker containers:
    
    ```bash
    docker build -t netflix .
    docker run -d --name netflix -p 8081:80 netflix:latest
    
    # to delete
    docker stop <containerid>
    docker rmi -f netflix
    ```

It will show an error cause you need an API key.

**Step 4: Get the API Key:**

- Open a web browser and navigate to TMDB (The Movie Database) website.
- Click on "Login" and create an account.
- Once logged in, go to your profile and select "Settings."
- Click on "API" from the left-side panel.
- Create a new API key by clicking "Create" and accepting the terms and conditions.
- Provide the required basic details and click "Submit."
- You will receive your TMDB API key.

Now recreate the Docker image with your API key:
```bash
docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .
```

### Phase 2: Security

**Install SonarQube and Trivy:**

- Install SonarQube and Trivy on the EC2 instance to scan for vulnerabilities.
    
    ```bash
    docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
    ```

    To access: `publicIP:9000` (by default username & password is admin)

    To install Trivy:
    ```bash
    sudo apt-get install wget apt-transport-https gnupg lsb-release
    wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
    echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
    sudo apt-get update
    sudo apt-get install trivy        
    ```
    
    To scan image using Trivy:
    ```bash
    trivy image <imageid>
    ```

### Phase 3: CI/CD Setup

**Install Jenkins for Automation:**

- Install Jenkins on the EC2 instance to automate deployment:
    
    ```bash
    sudo apt update
    sudo apt install fontconfig openjdk-17-jre
    java -version
    # Output should include:
    # openjdk version "17.0.8" 2023-07-18
    # OpenJDK Runtime Environment (build 17.0.8+7-Debian-1deb12u1)
    # OpenJDK 64-Bit Server VM (build 17.0.8+7-Debian-1deb12u1, mixed mode, sharing)
    
    sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
    echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
    sudo apt-get update
    sudo apt-get install jenkins
    sudo systemctl start jenkins
    sudo systemctl enable jenkins
    ```

- Access Jenkins in a web browser using the public IP of your EC2 instance: `publicIp:8080`

**Install Necessary Plugins in Jenkins:**

- Go to Manage Jenkins → Plugins → Available Plugins and install:
  - Eclipse Temurin Installer (Install without restart)
  - SonarQube Scanner (Install without restart)
  - NodeJs Plugin (Install Without restart)
  - Email Extension Plugin

**Configure Java and Nodejs in Global Tool Configuration:**

- Go to Manage Jenkins → Tools → Install JDK(17) and NodeJs(16)→ Click on Apply and Save

**SonarQube:**

- Create the token in SonarQube and add it to Jenkins under Manage Jenkins → Credentials → Add Secret Text.

**Configure CI/CD Pipeline in Jenkins:**

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/N4si/DevSecOps-Project.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
        }
        stage("quality gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
    }
}
```

### Phase 4: Monitoring

**Install Prometheus and Grafana:**

- Set up Prometheus and Grafana to monitor your application.

**Phase 5: Notification**

**Implement Notification Services:**

- Set up email notifications in Jenkins or other notification mechanisms.

### Phase 6: Kubernetes

**Create Kubernetes Cluster with Nodegroups**

**Monitor Kubernetes with Prometheus**

**Install Node Exporter using Helm**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
kubectl create namespace prometheus-node-exporter
helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter --namespace prometheus-node-exporter
```

**Add a Job to Scrape Metrics on nodeip:9001/metrics in prometheus.yml:**

```yaml
  - job_name: 'Netflix'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['node1Ip:9100']
```

### Deploy Application with ArgoCD

**Install ArgoCD:**

Follow the [EKS Workshop](https://archive.eksworkshop.com/intermediate/290_argocd/install/) documentation.

**Set Your GitHub Repository as a Source:**

Configure the connection to your repository and define the source for your ArgoCD application.

**Create an ArgoCD Application:**

- Set the application name, destination, project, source, and sync policy.

**Access your Application:**

Open `NodeIP:30007` in your browser.

### Phase 7: Cleanup

**Cleanup AWS EC2 Instances:**

- Terminate AWS EC2 instances that are no longer needed.
```

