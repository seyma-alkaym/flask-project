pipeline {
    agent any

    stages {
        stage('Git Cloning') {
            steps {
                echo 'Cloning git repo'
                git url: 'https://github.com/seyma-alkaym/flask-project.git', branch: 'main'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
                    sh '''
                    export PATH=$PATH:/opt/sonar-scanner-6.2.1.4610-linux-x64/bin
                    echo $PATH
                    echo "Checking for sonar-scanner"
                    command -v sonar-scanner || { echo "sonar-scanner not found"; exit 1; }
                    sonar-scanner \
                    -Dsonar.projectKey=flask-project \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=http://localhost:9000 \
                    -Dsonar.login=${SONAR_TOKEN}
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image'
                sh 'docker build -t flask-project .'
            }
        }


        stage('Update TRIVY DB') {
            steps {
                echo 'Updating TRIVY vulnerability database'
                sh 'trivy image flask-project:latest'
            }
        }

        stage('TRIVY Security Scan') {
            steps {
                echo 'Running TRIVY security scan'
                sh 'trivy image --skip-db-update flask-project:latest'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo 'Pushing Docker image to Docker Hub'
                withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh '''
                    echo "${PASS}" | docker login --username "${USER}" --password-stdin
                    docker tag flask-project ${USER}/flask-project:latest
                    docker push ${USER}/flask-project:latest
                    '''
                }
            }
        }

        stage('Integrate Remote Kubernetes with Jenkins') {
            steps {
                withCredentials([string(credentialsId: 'SECRET_TOKEN', variable: 'KUBE_TOKEN')]) {
                    sh '''
                    if ! command -v kubectl &> /dev/null; then
                        curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"
                        chmod +x kubectl
                    fi
                    export KUBECONFIG=$(mktemp)
                    ./kubectl config set-cluster minikube --server=https://172.31.234.40:8443 --insecure-skip-tls-verify=true
                    ./kubectl config set-credentials jenkins --token=${KUBE_TOKEN}
                    ./kubectl config set-context default --cluster=minikube --user=jenkins --namespace=default
                    ./kubectl config use-context default
                    ./kubectl get nodes
                    ./kubectl apply -f service.yaml
                    ./kubectl apply -f deployment.yaml
                    '''
                }
            }
        }
    }
}
