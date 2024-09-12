pipeline {
    agent any
    
    tools {
        nodejs 'node22'
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('Docker_id')
        DOCKER_IMAGE = "manoj3003/database:${BUILD_NUMBER}"
        OLD_IMAGE_TAG_PATTERN = "manoj3003/database:*"  // Pattern to match old images
        GIT_REPO_NAME = "swiggy-nodejs-devops-project"
        GIT_USER_NAME = "manoj7894"
        DEPLOYMENT_FILE_PATH = "Kubernetes/deployment.yml"  // Adjust the path if necessary
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Cleanup Old Docker Images') {
            steps {
                script {
                    echo "Cleaning up old Docker images..."
                    
                    def oldImages = sh(script: "docker images --format '{{.Repository}}:{{.Tag}}' | grep '${OLD_IMAGE_TAG_PATTERN}'", returnStdout: true).trim()
                    
                    if (oldImages) {
                        oldImages.split('\n').each { image ->
                            if (image != DOCKER_IMAGE) {
                                echo "Attempting to remove old image ${image}"
                                sh """
                                    if docker images -q ${image} > /dev/null 2>&1; then
                                        docker rmi -f ${image} || echo 'Failed to remove image ${image} - might be in use or other error.'
                                    else
                                        echo 'Image ${image} does not exist.'
                                    fi
                                """
                            }
                        }
                    } else {
                        echo "No old images found matching pattern ${OLD_IMAGE_TAG_PATTERN}"
                    }
                }
            }
        }
        stage('Cleanup Old Trivy Reports') {
            steps {
                script {
                    echo "Cleaning up old Trivy reports..."
                    sh '''
                        rm -f trivy.txt
                        rm -f fs-report.html
                    '''
                }
            }
        }
        
        stage('GetCode') {
            steps {
                git branch: 'main', url: 'https://github.com/manoj7894/swiggy-nodejs-devops-project.git'
            }
        }
        
        stage('Install Package Dependencies') {
            steps {
                sh "npm install"
            }
        }
        
        stage('Check for Tests') {
            steps {
                script {
                    def testScriptExists = sh(script: "grep -q '\"test\":' package.json", returnStatus: true) == 0
                    env.TEST_SCRIPT_EXISTS = testScriptExists ? 'true' : 'false'
                }
            }
        }
        
        /*
        stage('unit test') {
            when {
                expression { env.TEST_SCRIPT_EXISTS == 'true' }
            }
            steps {
                script {
                    try {
                        sh "npm test"
                        currentBuild.result = 'SUCCESS'
                    } catch (Exception e) {
                        echo "Unit tests failed. Continuing with the pipeline."
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        } */
        
        stage('OWASP SCAN') {
            when {
                expression {
                    // Check if pom.xml exists
                    return fileExists('pom.xml')
                }
            }
            steps {
                dependencyCheck additionalArguments: '', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        
        stage("Trivy filesystem scan") {
            steps {
                script {
                    sh "trivy fs --format table -o fs-report.html ."
                }
            }
        }
        
        stage('Build') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'Docker_id', toolName: 'Docker') {
                        sh "docker build -t ${DOCKER_IMAGE} ."
                    }
                }
            }
        }
        
        stage("Trivy image scan") {
            steps {
                script {
                    sh "trivy image ${DOCKER_IMAGE} > trivy.txt"
                }
            }
        }
        
        stage("SonarQube") {
            steps {
                withSonarQubeEnv('Sonar_Install') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Campground \
                    -Dsonar.projectKey=Campground '''
                }
            }
        }

        stage('Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'Docker_id', toolName: 'Docker') {
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Update Deployment File') {
            steps {
                withCredentials([string(credentialsId: 'github_id', variable: 'GITHUB_TOKEN')]) {
                    script {
                        sh '''
                            git config user.email "manojvarmapotthuri3003@gmail.com"
                            git config user.name "Manojvarma Potthuri"
                        '''

                        sh 'git checkout main'
                        sh 'cat ${DEPLOYMENT_FILE_PATH}'

                        sh '''
                            sed -i "s|image: manoj3003/database:[^[:space:]]*|image: manoj3003/database:${BUILD_NUMBER}|g" ${DEPLOYMENT_FILE_PATH}
                        '''

                        sh '''
                            if git diff --quiet; then
                                echo "No changes detected in ${DEPLOYMENT_FILE_PATH}"
                            else
                                git add ${DEPLOYMENT_FILE_PATH}
                                git commit -m "Update deployment image to version ${BUILD_NUMBER}"
                                git push -f https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git HEAD:main
                            fi
                        '''
                    }
                }
            }
        }
        
        stage('K8S-Deploy') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: ' eks-1', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://05FF96831A57B61D6882E866196EB072.sk1.ap-south-1.eks.amazonaws.com') {
                sh "kubectl apply -f Kubernetes/deployment.yml"
                sh "kubectl apply -f Kubernetes/service.yml"
                sleep 20
                }
            }
        }
        
         stage('Verify the Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: ' eks-1', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://05FF96831A57B61D6882E866196EB072.sk1.ap-south-1.eks.amazonaws.com') {
                sh "kubectl get pods"
                sh "kubectl get svc"
                }
            }
        }
    }
    
    

    post {
        always {
            emailext attachLog: true,
                subject: "Pipeline Status: ${BUILD_NUMBER}",
                body: '''<html>
                           <body>
                              <p>Build Status: ${BUILD_STATUS}</p>
                              <p>Build Number: ${BUILD_NUMBER}</p>
                              <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                           </body>
                        </html>''',
                to: 'manojvarmapotthutri@gmail.com',
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                attachmentsPattern: 'trivy.txt',
                mimeType: 'text/html'
        }
    }
}
