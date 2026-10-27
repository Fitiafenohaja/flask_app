pipeline {

    agent any

    stages {

        stage('Test Flask') {
            steps {
                script {
                    docker.image('python:3.11').inside('-u root') {
                        sh 'pip install -r requirements.txt'
                        sh 'python test.py'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t flask_hello:latest .'
            }
        }

        stage('Push Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub_credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker tag flask_hello:latest fitiafenohajaun/flask_hello_jenkins:latest
                    docker push fitiafenohajaun/flask_hello_jenkins:latest
                    '''
                }
            }
        }

        stage('Deploy Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    script {
                        // Réseau "minikube" pour joindre l'API du cluster (192.168.49.2)
                        docker.image('alpine/k8s:1.31.4').inside("-u root --network minikube --entrypoint=''") {
                            sh '''
                            kubectl apply -f k8s/deployment.yml
                            kubectl apply -f k8s/service.yml
                            # Le tag reste "latest" : on force le redémarrage pour récupérer la nouvelle image
                            kubectl rollout restart deployment/flask-hello-deployment
                            kubectl rollout status deployment/flask-hello-deployment --timeout=120s
                            '''
                        }
                    }
                }
            }
        }
    }
}