pipeline {
    // Defines that this pipeline can run on any available agent (worker node)
    agent any

    // Configuration to automatically trigger the pipeline when code is pushed to GitHub
    triggers {
        githubPush()
    }

    // Global variables used throughout the pipeline
    environment {
        DOCKER_HUB_USER = 'azoooz'
        // We use the Jenkins Build Number as an Immutable Tag for security and easy rollbacks
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
        GIT_REPO_URL    = "https://github.com/MuhammedAhmedAbdulaziz/microservices-ecommerce.git"
    }

    stages {
        // Stage 1: Pull the latest source code from the repository
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: "${env.GIT_REPO_URL}"
            }
        }

        // Stage 2: Optimized Build. It builds only the services that actually changed.
        // Parallel execution saves time by running builds simultaneously.
        stage('Build & Push Microservices') {
            parallel {
                stage('Home Service') {
                    // Only run if changes are detected in this specific directory
                    when { changeset "home-service/**" }
                    steps {
                        script { buildAndPush("home-service") }
                    }
                }
                stage('About Service') {
                    when { changeset "about-service/**" }
                    steps {
                        script { buildAndPush("about-service") }
                    }
                }
                stage('Contact Service') {
                    when { changeset "contact-service/**" }
                    steps {
                        script { buildAndPush("contact-service") }
                    }
                }
                stage('Services Service') {
                    when { changeset "services-service/**" }
                    steps {
                        script { buildAndPush("services-service") }
                    }
                }
            }
        }

        // Stage 3: The GitOps Bridge. Updates the Manifests to trigger ArgoCD.
        stage('Update Helm Values (GitOps)') {
            steps {
                script {
                    // Uses Jenkins Credentials to securely push changes back to GitHub
                    withCredentials([usernamePassword(credentialsId: 'github-token', passwordVariable: 'GIT_PASS', usernameVariable: 'GIT_USER')]) {
                        sh """
                            git config user.email "jenkins@example.com"
                            git config user.name "Jenkins CI"
                            
                            # Use 'sed' to update the image tag in the Helm values.yaml file
                            sed -i 's/tag: .*/tag: "${IMAGE_TAG}"/g' ecommerce-chart/values.yaml
                            
                            git add ecommerce-chart/values.yaml
                            # [skip ci] is used to prevent an infinite loop of Jenkins builds
                            git commit -m "Update image tag to ${IMAGE_TAG} [skip ci]"
                            git push https://${GIT_USER}:${GIT_PASS}@github.com/MuhammedAhmedAbdulaziz/microservices-ecommerce.git main
                        """
                    }
                }
            }
        }
    }

    // Post-build actions for notifications and cleanup
    post {
        success {
            echo "CI/CD Pipeline finished successfully! ArgoCD will now detect the change and sync the cluster."
        }
        always {
            // Cleans up local Docker images to save disk space on the Jenkins server
            sh "docker system prune -f"
        }
    }
}

/**
 * Shared function to handle the Docker Build, Tag, and Push logic
 */
def buildAndPush(serviceName) {
    dir(serviceName) {
        withCredentials([usernamePassword(credentialsId: 'docker-credi', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
            sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
            
            // Build and Push with the unique Build Number tag
            sh "docker build -t ${DOCKER_HUB_USER}/${serviceName}:${IMAGE_TAG} ."
            sh "docker push ${DOCKER_HUB_USER}/${serviceName}:${IMAGE_TAG}"
            
            // Also push as 'latest' for quick local testing
            sh "docker tag ${DOCKER_HUB_USER}/${serviceName}:${IMAGE_TAG} ${DOCKER_HUB_USER}/${serviceName}:latest"
            sh "docker push ${DOCKER_HUB_USER}/${serviceName}:latest"
        }
    }
}