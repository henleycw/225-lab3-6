pipeline {
    agent any 

    environment {
        DOCKER_CREDENTIALS_ID = 'roseaw-dockerhub'
        DOCKER_IMAGE = 'cithit/roseaw'                                                                    //<------change this
        IMAGE_TAG = "build-${BUILD_NUMBER}"
        GITHUB_URL = 'https://github.com/miamioh-cit/lab3-6.git'                                          //<------change this
        KUBECONFIG = credentials('roseaw-225')                                                         //<------change this
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']],
                          userRemoteConfigs: [[url: "${GITHUB_URL}"]]])
            }
        }

        stage('Lint HTML') {
            steps {
                sh 'npm install htmlhint --save-dev'
                sh 'npx htmlhint *.html'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://registry-1.docker.io', 'roseaw-dockerhub') {
                        docker.build("${DOCKER_IMAGE}:${IMAGE_TAG}", "-f Dockerfile.build .")
                    }
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CREDENTIALS_ID}") {
                        docker.image("${DOCKER_IMAGE}:${IMAGE_TAG}").push()
                    }
                }
            }
        }

        stage('Deploy to Dev Environment') {
            steps {
                script {
                    // Set up Kubernetes configuration using the specified KUBECONFIG
                    def kubeConfig = readFile(KUBECONFIG)
                    // Update deployment-dev.yaml to use the new image tag
                    sh "sed -i 's|${DOCKER_IMAGE}:latest|${DOCKER_IMAGE}:${IMAGE_TAG}|' deployment-dev.yaml"
                    sh "kubectl apply -f deployment-dev.yaml"
                }
            }
        }

        stage("Run Acceptance Tests") {
            steps {
                script {
                    sh 'docker stop qa-tests || true'
                    sh 'docker rm qa-tests || true'
                    sh 'docker build -t qa-tests -f Dockerfile.test .'
                    sh 'docker run qa-tests'
                    sh 'docker stop qa-tests || true'
                    sh 'docker rm qa-tests || true'
                }
            }
        }

        stage('Deploy to Prod Environment') {
            steps {
                script {
                    // Set up Kubernetes configuration using the specified KUBECONFIG
                    //sh "ls -la"
                    sh "sed -i 's|${DOCKER_IMAGE}:latest|${DOCKER_IMAGE}:${IMAGE_TAG}|' deployment-prod.yaml"
                    sh "cd .."
                    sh "kubectl apply -f deployment-prod.yaml"
                }
            }
        }
        stage('Check Kubernetes Cluster') {
            steps {
                script {
                    sh "kubectl get all"
                }
            }
        }
    }
    post {
        
        success {
            slackSend color: "good", message: "Build Completed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
        unstable {
            slackSend color: "warning", message: "Build Unstable: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
        failure {
            slackSend color: "danger", message: "Build Failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
    }
}

