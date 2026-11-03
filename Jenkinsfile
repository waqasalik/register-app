pipeline {
    agent { label "Jenkins-Agent" }
    
    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'The image tag to deploy')
    }

    environment {
        APP_NAME    = "accounting-app"
        DOCKER_USER = "waqaszai"
        REPO_URL    = "https://github.com/waqasalik/register-app"
        CI_REPO_URL = "https://github.com/waqasalik/Accounting-app.git"
        IMAGE_NAME  = "${DOCKER_USER}/${APP_NAME}:${params.IMAGE_TAG}"
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        // ==========================================
        // CI PIPELINE (Build & Scan Application)
        // ==========================================
        
        stage("Checkout App Code") {
            steps {
                git branch: 'main', credentialsId: 'github', url: "${CI_REPO_URL}"
            }
        }

        // [Insert the Build and Test stages here]

        stage("Build Docker Image") {
            steps {
                sh """
                    docker build -t ${IMAGE_NAME} .
                """
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                    # THE FIX: Redirect Trivy's temp directory to the workspace to avoid "no space left on device"
                    export TMPDIR="$(pwd)/.trivytmp"
                    mkdir -p "$TMPDIR"
                    
                    trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_NAME}
                '''
            }
        }

        stage("Push Docker Image") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                    sh """
                        echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin
                        docker push ${IMAGE_NAME}
                    """
                }
            }
        }

        // ==========================================
        // CD PIPELINE (GitOps Manifest Update)
        // ==========================================
        
        stage("Update GitOps Manifest") {
            steps {
                sh """
                    echo "Updating deployment to tag: ${params.IMAGE_TAG}"
                    
                    # Clone the GitOps repo into a sub-folder
                    git clone https://github.com/waqasalik/register-app.git gitops-repo
                    cd gitops-repo

                    # Update the image tag
                    sed -i 's|image: ${DOCKER_USER}/${APP_NAME}:.*|image: ${DOCKER_USER}/${APP_NAME}:${params.IMAGE_TAG}|g' deployment.yaml
                    
                    # Show the change
                    cat deployment.yaml
                """
            }
        }

        stage("Push the changed deployment file to Git") {
            steps {
                sh """
                    cd gitops-repo
                    
                    # Configure Git user
                    git config --global user.name "waqasalik"
                    git config --global user.email "waqasalikhan086@gmail.com" 
                    
                    # Commit the change
                    git add deployment.yaml
                    git commit -m "Update image tag to ${params.IMAGE_TAG} [ci skip]" || echo "No changes to commit"
                """
                
                // Use the credential to push securely
                withCredentials([gitUsernamePassword(credentialsId: 'github', gitToolName: 'Default')]) {
                    sh """
                        cd gitops-repo
                        git push origin main
                    """
                }
            }
        }
    }
}
