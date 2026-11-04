pipeline {
    agent { label "Jenkins-Agent" }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: '', description: 'Leave empty to use BUILD_NUMBER')
    }

    environment {
        APP_NAME        = "accounting-app"
        DOCKER_USER     = "waqaszai"
        REPO_URL        = "https://github.com/waqasalik/register-app.git"
        CI_REPO_URL     = "https://github.com/waqasalik/Accounting-app.git"

        FINAL_TAG       = "${params.IMAGE_TAG ?: env.BUILD_NUMBER}"
        IMAGE_NAME      = "${DOCKER_USER}/${APP_NAME}:${FINAL_TAG}"
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
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage("Trivy Scan") {
            steps {
                sh """
                    mkdir -p /home/ubuntu/trivy-cache
                    docker run --rm \\
                        -v /var/run/docker.sock:/var/run/docker.sock \\
                        -v /home/ubuntu/trivy-cache:/root/.cache/ \\
                        aquasec/trivy:latest image \\
                        --scanners vuln --ignore-unfixed \\
                        --exit-code 1 --severity HIGH,CRITICAL \\
                        ${IMAGE_NAME}
                """
            }
        }

        stage("Push Docker Image") {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    passwordVariable: 'DOCKER_PASS',
                    usernameVariable: 'DOCKER_USER_CRED'
                )]) {
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

                    rm -rf gitops-repo
                    git clone ${REPO_URL} gitops-repo
                    cd gitops-repo

                    sed -i 's|image: ${DOCKER_USER}/${APP_NAME}:.*|image: ${DOCKER_USER}/${APP_NAME}:${FINAL_TAG}|g' deployment.yaml
                    cat deployment.yaml
                """
            }
        }

        stage("Push the changed deployment file to Git") {
            steps {
                sh """
                    cd gitops-repo
                    git config user.name  "waqasalik"
                    git config user.email "waqasalikhan086@gmail.com"
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

    }  // <-- closes stages

    post {
        always {
            sh 'docker logout || true'
        }
        success {
            echo "Successfully deployed ${IMAGE_NAME}"
        }
        failure {
            echo "Pipeline failed while building ${IMAGE_NAME}"
        }
    }

}  // <-- closes pipeline
