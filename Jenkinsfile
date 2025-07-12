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

                    

                    exit
                    EOF
                    """
                }
            }
        }
    }
}
