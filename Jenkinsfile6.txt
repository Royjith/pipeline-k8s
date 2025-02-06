pipeline {
     agent { label 'node-1' } // Set the agent to use node-1

    environment {
        DOCKER_IMAGE = 'my-app'               // Docker image name
        DOCKER_TAG = 'latest-v5'                 // Docker tag
        DOCKER_HUB_REPO = 'royjith/pikube'    // Docker Hub repository
        DOCKER_HUB_CREDENTIALS_ID = 'dockerhub'  // Docker Hub credentials ID
        KUBE_CONFIG = '/tmp/kubeconfig'  // Path to the kubeconfig file or use Jenkins Kubernetes plugin credentials
        DEPLOYMENT_NAME = 'pipeline-deployment'
        NAMESPACE = 'default'  // Kubernetes namespace to deploy to
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from Git...'
                git branch: 'main', credentialsId: 'dockerhub', url: 'https://github.com/Royjith/docker.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Set default tag to 'latest' if DOCKER_TAG is not defined
                    def tag = "${DOCKER_TAG ?: 'latest-v5'}"
                    echo "Building Docker image with tag: ${tag}..."
                    // Build the Docker image with the determined tag
                    def buildResult = sh(script: "docker build -t ${DOCKER_HUB_REPO}:${tag} .", returnStatus: true)
            
                    if (buildResult != 0) {
                        error 'Docker build failed!'  // Explicitly fail if Docker build fails
                    }
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    echo 'Running Trivy security scan on the Docker image...'

                    // Run Trivy scan for vulnerabilities in the Docker image
                    def scanResult = sh(script: "trivy image ${DOCKER_HUB_REPO}:${DOCKER_TAG}", returnStatus: true)

                    // Fail the build if vulnerabilities are found (returnStatus != 0 means issues were detected)
                    if (scanResult != 0) {
                        error 'Trivy scan found vulnerabilities in the Docker image!'
                    } else {
                        echo 'Trivy scan passed: No vulnerabilities found.'
                    }
                }
            }
        }

        stage('Push Image to DockerHub') {
            steps {
                input message: 'Approve Deployment?', ok: 'Deploy'  // Manual approval for deployment
                script {
                    echo 'Pushing Docker image to DockerHub...'

                    try {
                        // Manually login to Docker Hub using the credentials
                        withCredentials([usernamePassword(credentialsId: "${DOCKER_HUB_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                            sh '''
                                echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
                            '''
                        }

                        // Push the Docker image to Docker Hub
                        sh "docker push ${DOCKER_HUB_REPO}:${DOCKER_TAG}"

                    } catch (Exception e) {
                        error "Docker push failed: ${e.message}"  // Explicitly fail if push fails
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                branch 'main'  // Only deploy on the 'main' branch
            }
            steps {
                input message: 'Approve Kubernetes Deployment?', ok: 'Deploy'  // Manual approval before deployment
                script {
                    echo 'Deploying Docker image to Kubernetes...'

                    try {
                        // Set the kubeconfig file (for accessing the Kubernetes cluster)
                        withCredentials([file(credentialsId: 'pikube', variable: 'KUBECONFIG_FILE')]) {

                            // Define the path to your deployment.yaml file in the repository
                            def deploymentFile = 'deployment.yaml'  // Adjust path if necessary

                            // Verify if the deployment.yaml exists in the workspace
                            sh 'ls -al ${deploymentFile}'

                            // Update the Docker image in the deployment.yaml with the newly pushed image tag
                            echo "Updating Docker image in the deployment.yaml to ${DOCKER_HUB_REPO}:${DOCKER_TAG}"
                            sh """
                                sed -i 's|image: .*|image: ${DOCKER_HUB_REPO}:${DOCKER_TAG}|g' ${deploymentFile}
                            """

                            // Apply the updated deployment.yaml using kubectl
                            echo 'Applying the updated deployment.yaml to the Kubernetes cluster...'
                            sh """
                                export KUBECONFIG=$KUBECONFIG_FILE
                                kubectl apply -f ${deploymentFile} --namespace=${NAMESPACE}
                            """
                        }
                    } catch (Exception e) {
                        error "Kubernetes deployment failed: ${e.message}"  // Explicitly fail if Kubernetes deployment fails
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()  // Clean workspace after pipeline execution
        }
        success {
            echo 'Pipeline executed successfully!'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
