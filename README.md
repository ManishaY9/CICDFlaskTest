# CI/CD Pipeline using Jenkins and GitHub Actions

This guide provides comprehensive instructions to set up a CI/CD pipeline using **Jenkins** and **GitHub Actions** for a Python web application. The pipeline includes build, test, and deployment stages to an AWS EC2 instance.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Setup Jenkins](#setup-jenkins)
3. [Configure GitHub Actions](#configure-github-actions)
4. [CI/CD Workflow](#cicd-workflow)
5. [Deployment to AWS EC2](#deployment-to-aws-ec2)
6. [Screenshots and Attachments](#screenshots-and-attachments)
7. [Troubleshooting](#troubleshooting)
8. [Conclusion](#conclusion)

---

## Prerequisites
- A GitHub repository containing the Python web application.
- An AWS EC2 instance with SSH access.
- Jenkins installed and running (using Docker or a dedicated server).
- GitHub Actions enabled on your repository.
- Python installed on Jenkins and EC2.

---

## Setup Jenkins

### 1. Install Jenkins via Docker

```bash
docker network create jenkins
```


#### Start Docker-in-Docker (dind)

```bash
docker run --name jenkins-docker --rm --detach \
  --privileged --network jenkins --network-alias docker \
  --env DOCKER_TLS_CERTDIR=/certs \
  --volume jenkins-docker-certs:/certs/client \
  --volume jenkins-data:/var/jenkins_home \
  --publish 2376:2376 \
  docker:dind
```

#### Create Jenkins Dockerfile

```Dockerfile
FROM jenkins/jenkins:2.492.2-jdk17
USER root
RUN apt-get update && apt-get install -y lsb-release
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
  https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
  https://download.docker.com/linux/debian $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean docker-workflow"
```

#### Build & Run Jenkins Container

```bash
docker build -t myjenkins-blueocean:2.492.2-1 .
docker run --name jenkins-blueocean --restart=on-failure --detach \
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  --publish 8080:8080 --publish 50000:50000 myjenkins-blueocean:2.492.2-1
```


### 2. Required Jenkins Plugins

* Git Plugin
* Pipeline Plugin
* SSH Pipeline Steps
* GitHub Integration Plugin


### 3. Configure GitHub Webhook

1. Go to GitHub → Repository → Settings → Webhooks
2. Add webhook URL: `http://<JENKINS-IP>:8080/github-webhook/`
3. Set content type: `application/json`
4. Trigger on `push` events


### 4. Create a Jenkins Pipeline
1. Navigate to **Jenkins Dashboard → New Item → Pipeline**
2. Add the following pipeline script:

```groovy
pipeline {
    agent any
    environment {
        APP_DIR = "flask_app"
        SSH_KEY = "/var/jenkins_home/.ssh/id_rsa"
        EC2_USER = "ubuntu"
        EC2_IP = "13.57.8.246"
    }
    stages {
        stage('Checkout Code') {
            steps {
                dir("${WORKSPACE}/${APP_DIR}") {
                    git branch: 'main',
                        url: 'https://github.com/ManishaY9/CICDFlaskTest.git'
                }
            }
        }
        stage('Build') {
            steps {
                dir("${WORKSPACE}/${APP_DIR}") {
                    sh '''
                    apt update && apt install -y python3-venv python3-pip
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    '''
                }
            }
        }
        stage('Test') {
            steps {
                dir("${WORKSPACE}/${APP_DIR}") {
                    sh '''
                    . venv/bin/activate
                    pip install pytest
                    pytest || true
                    '''
                }
            }
        }
        stage('Deploy to EC2') {
            steps {
                script {
                    sh """
                    ssh -o StrictHostKeyChecking=no -i ${SSH_KEY} ${EC2_USER}@${EC2_IP} <<EOF
                    set -e  # Stop on error

                    # Create application directory if it doesn't exist
                    mkdir -p ${APP_DIR}

                    # Navigate to application directory
                    cd ${APP_DIR} || exit 1

                    # Clone the repository if it doesn't exist
                    if [ ! -d ".git" ]; then
                        git clone https://github.com/ManishaY9/CICDFlaskTest.git .
                    fi

                    # Pull the latest code from the repository
                    git pull origin main

                    # Set up virtual environment and install dependencies
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt

                    # Run the Flask application
                    nohup python3 -m app &

                    exit
                    EOF
                    """
                }
            }
        }
    }
}
```

---

## Configure GitHub Actions

### 1. Workflow File: `.github/workflows/ci-cd.yml`

```yaml
name: Flask CI/CD Pipeline

on:
  push:
    branches:
      - main
      - staging
  pull_request:
    branches:
      - main
      - staging

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: 3.9

      - name: Install Dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run Tests
        run: python -m unittest discover

  deploy-staging:
    needs: test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Deploy to Staging Server
        env:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          STAGING_HOST: ${{ secrets.STAGING_HOST }}
          STAGING_USER: ${{ secrets.STAGING_USER }}
      
        run: |
          echo "$SSH_PRIVATE_KEY" > deploy_key.pem
          chmod 600 deploy_key.pem

          ssh -o StrictHostKeyChecking=no -i deploy_key.pem $STAGING_USER@$STAGING_HOST << 'EOF'
            set -e  # Exit on error
            
            # Ensure project directory exists
            mkdir -p /home/ubuntu/FlaskTest
            cd /home/ubuntu/FlaskTest

            # Clone repo if it doesn't exist
            if [ ! -d ".git" ]; then
              git clone git@github.com:your-user/your-repo.git .
            fi

            # Ensure correct branch
            git fetch origin
            if git rev-parse --verify staging; then
              git checkout staging
            else
              git checkout -b staging
            fi
            git pull origin staging

            # Ensure virtual environment exists
            if [ ! -d "venv" ]; then
              python3 -m venv venv
            fi
            source venv/bin/activate

            # Check if requirements.txt exists before installing dependencies
            if [ -f "requirements.txt" ]; then
              pip install -r requirements.txt
            else
              echo "ERROR: requirements.txt not found!"
              exit 1
            fi

            # Restart Flask application
            if systemctl list-units --full -all | grep -Fq "flaskapp.service"; then
              sudo systemctl restart flaskapp
            else
              echo "Warning: flaskapp.service not found. Ensure it's set up."
            fi
          EOF
 ```

### 2. Add GitHub Secrets

* `SSH_PRIVATE_KEY`: Private key for EC2 access
* `STAGING_HOST`: EC2 IP
* `STAGING_USER`: EC2 username (`ubuntu`)

---

## 🔁 CI/CD Workflow Overview

1. Developer pushes to `main` or `staging`
2. GitHub Actions triggers test and optional deploy
3. Jenkins picks up webhook, pulls code, builds, tests, and deploys
4. Flask app is updated on EC2 instance 

---

## ☁️ AWS EC2 Deployment

### 1. Connect to EC2

```bash
ssh -i your-key.pem ubuntu@<ec2-ip>
```

### 2. Install Required Packages

```bash
sudo apt update && sudo apt install python3-pip git -y
```

### 3. Clone Repo and Setup Systemd

```bash
git clone https://github.com/your-repo.git /var/www/app
```
### 4. Setup a systemd service:
Add:

```ini
[Unit]
Description=My Python App
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/var/www/app
ExecStart=/usr/bin/python3 /var/www/app/app.py
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp
```

---

## 🛠 Troubleshooting

* **Permission Denied (SSH):**

  ```bash
  chmod 400 your-key.pem
  ```

* **Jenkins Webhook Not Triggering:**
  Check GitHub Webhook status and Jenkins logs.

* **GitHub Actions SSH Fails:**
  Ensure SSH key and EC2 IP are correctly set in GitHub Secrets.

---

## Conclusion
This guide provides a fully automated CI/CD pipeline integrating Jenkins and GitHub Actions for deploying a Python web application to AWS EC2. 🚀

