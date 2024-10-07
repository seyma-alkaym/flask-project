pipeline {
    agent any

    stages {
        stage('Git Cloning') {
            steps {
                echo 'Cloning git repo'
                git url: 'https://github.com/seyma-alkaym/flask-project.git',  branch: 'main'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                sh """
                    mvn sonar:sonar \
                    -Dsonar.projectKey=flask-project \
                    -Dsonar.host.url='http://localhost:9000' \
                    -Dsonar.login=credentials('SONAR_TOKEN')
                """
            }
        }
        stage('Build Docker Image') {
            steps {
                echo 'Building the image'
                sh 'docker build -t flask-project .'
            }
        }
        stage('Push to Docker Hub') {
            steps {
                echo 'Pushing to Docker Hub'
                withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh '''
                    echo "${PASS}" | docker login --username "${USER}" --password-stdin
                    docker tag flask-project ${USER}/flask-project:latest
                    docker push ${USER}/flask-project:latest
                    '''
                }
            }
        }
        stage("TRIVY") {
            steps {
                sh "trivy image flask-project:latest"
            }
        }
        stage('Integrate Remote kubernetess with Jenkins') {
            steps {
                withCredentials([string(credentialsId: 'SECRET_TOKEN', variable: 'KUBE_TOKEN')]) {
                    sh '''
                    curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"
                    chmod u+x ./kubectl
                    export KUBECONFIG=$(mktemp)
                    ./kubectl config set-cluster minikube --server=https://172.20.201.241:8443 --insecure-skip-tls-verify=true
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
