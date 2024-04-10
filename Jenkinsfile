pipeline {
    agent any 

    environment {
        DOCKER_CREDENTIALS_ID = 'roseaw-dockerhub'  
        DOCKER_IMAGE = 'cithit/roseaw'                                   //<-----change this to your MiamiID!
        IMAGE_TAG = "build-${BUILD_NUMBER}"
        GITHUB_URL = 'https://github.com/miamioh-cit/225-lab4-3.git'     //<-----change this to match this new repository!
        KUBECONFIG = credentials('roseaw-225')                           //<-----change this to match your kubernetes credentials (MiamiID-225)! 
    }

    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                checkout([$class: 'GitSCM', branches: [[name: '*/main']],
                          userRemoteConfigs: [[url: "${GITHUB_URL}"]]])
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${IMAGE_TAG}")
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
                    // This sets up the Kubernetes configuration using the specified KUBECONFIG
                    def kubeConfig = readFile(KUBECONFIG)
                    // This updates the deployment-dev.yaml to use the new image tag
                    sh "sed -i 's|${DOCKER_IMAGE}:latest|${DOCKER_IMAGE}:${IMAGE_TAG}|' deployment-dev.yaml"
                    sh "kubectl apply -f deployment-dev.yaml"
                }
            }
        }

         stage("Prepare Gauntlt Container") {
            steps {
                // Pull your custom Gauntlt Docker image
                sh 'docker pull cithit/gauntlt:build-4'
                
                // Check if a container named 'gauntlt-runner' already exists, and remove it if it does
                sh 'docker rm -f gauntlt-runner || true'

                // Create a Docker container without starting it
                sh 'docker create --name gauntlt-runner -v \$(pwd)/test-files:/gauntlt-tests cithit/gauntlt:build-4'
            }
        }
        
        stage("Copy Test Files") {
            steps {
                // Copy test files to the container - here we assume files are already where they need to be via volume
                echo 'Files are mounted and ready in gauntlt-runner'
            }
        }
        
        stage("Run Gauntlt Attacks") {
            steps {
                // Start the container and run Gauntlt
                sh 'docker start -a gauntlt-runner'
        
                // Execute Gauntlt attack
                sh 'docker exec gauntlt-runner gauntlt /gauntlt-tests/port.attack'
        
                // Optionally, you can stop and remove the container after the test
                sh 'docker rm -f gauntlt-runner'
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
            slackSend color: "warning", message: "Build Completed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
        failure {
            slackSend color: "danger", message: "Build Completed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
    }
}
