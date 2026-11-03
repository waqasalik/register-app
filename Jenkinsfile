pipeline {
    agent { label "Jenkins-Agent" }
    
    parameters {
        // Leave this blank. If triggered by another job, it will pass a value here.
        string(name: 'IMAGE_TAG', defaultValue: '', description: 'Leave empty to use BUILD_NUMBER')
    }

    environment {
        APP_NAME    = "accounting-app"
        DOCKER_USER = "waqaszai"
        REPO_URL    = "https://github.com/waqasalik/register-app"
        CI_REPO_URL = "https://github.com/waqasalik/Accounting-app.git"
        
        // FIX: If IMAGE_TAG is empty, fallback to the Build Number. Otherwise use IMAGE_TAG.
        FINAL_TAG   = "${params.IMAGE_TAG ?: env.BUILD_NUMBER}"
        IMAGE_NAME  = "${DOCKER_USER}/${APP_NAME}:${FINAL_TAG}"
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }
        
        stage("Checkout App Code") {
            steps {
                git branch: 'main', credentialsId: 'github', url: "${CI_REPO_URL}"
            }
        }

        stage("Build Docker Image") {
            steps {
                sh """
                    docker build -t ${IMAGE_NAME} .
                """
            }
        }
         stage('Trivy Scan') {
             steps {
                 sh """
            echo "Running Trivy scan via Docker..."
            mkdir -p trivy-cache
            docker run --rm \\
                -v /var/run/docker.sock:/var/run/docker.sock \\
                -v \$(pwd)/trivy-cache:/root/.cache/ \\
                aquasec/trivy:latest image --skip-java-db-update --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_NAME}
        """
    }
}
        stage("Push Docker Image") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER_CRED')]) {
                    sh """
                        echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER_CRED" --password-stdin
                        docker push ${IMAGE_NAME}
                    """
                }
            }
        }
        
        stage("Update GitOps Manifest") {
            steps {
                sh """
                    echo "Updating deployment to tag: ${FINAL_TAG}"
                    
                    # Clone the GitOps repo into a sub-folder
                    git clone https://github.com/waqasalik/register-app.git gitops-repo
                    cd gitops-repo

                    # Update the image tag
                    sed -i 's|image: ${DOCKER_USER}/${APP_NAME}:.*|image: ${DOCKER_USER}/${APP_NAME}:${FINAL_TAG}|g' deployment.yaml
                    
                    # Show the change
                    cat deployment.yaml
                """
            }
        }

        stage("Push the changed deployment file to Git") {
            steps {
                sh """
                    cd gitops-repo
                    git config --global user.name "waqasalik"
                    git config --global user.email "waqasalikhan086@gmail.com" 
                    git add deployment.yaml
                    git commit -m "Update image tag to ${FINAL_TAG} [ci skip]" || echo "No changes to commit"
                """
                
                withCredentials([gitUsernamePassword(credentialsId: 'github', gitToolName: 'Default')]) {
                    sh """
                        cd gitops-repo
                        git push origin main
                    """
                }
            }   
        } 

        stage('Trigger GitOps CD Pipeline') {
            steps {
                echo "Triggering downstream job: gitops-register-app-cd"
                build job: 'gitops-register-app-cd', 
                    wait: false, 
                    parameters: [
                        string(name: 'IMAGE_TAG', value: "${FINAL_TAG}")
                    ]
            }
        }
    }
}
