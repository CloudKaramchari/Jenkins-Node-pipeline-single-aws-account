pipeline {
    agent any

    parameters {
        choice(name: "DEPLOY_TO", choices: ["", "PREPROD", "PROD"], description: 'Select deployment environment')
        string(name: "BRANCHES", defaultValue: 'master', description: 'Please enter branch name')
        string(name: "gitAccount", defaultValue: '', description: 'GitHub account (organization/user) name')
        credentials(name: "credentialsId", description: 'Select Jenkins credentials for Git')
    }

    stages {
        stage('Pull Service Repo Name') {
            steps {
                script {
                    def repo_name = env.JOB_NAME
                    echo "Service Repo Name: ${repo_name}"
                }
            }
        }

        stage('Pull Git Repo') {
            steps {
                echo "Pulling code from GitHub repository..."
                git branch: "${params.BRANCHES}", 
                    credentialsId: "${params.credentialsId}", 
                    url: "https://github.com/${params.gitAccount}/${env.JOB_NAME}.git"
                echo "Repository pulled successfully."
            }
        }

        stage('Deployment') {
            parallel {
                stage("PREPROD") {
                    when { expression { params.DEPLOY_TO == "PREPROD" } }
                    stages {
                        stage('Update old PREPROD ECR version tag') {
                            steps {
                                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                    load "./jenkins/PREPROD/env.jenkins"
                                    echo "${AWS_ID}"
                                    echo "${DEPLOY_TO}"

                                    sh '''#!/bin/bash
                                    MANIFEST=$(/usr/local/bin/aws ecr batch-get-image --repository-name "${image}" --region ap-south-1 --image-ids imageTag=latest --query 'images[].imageManifest' --output text)
                                    /usr/local/bin/aws ecr put-image --repository-name "${image}" --region ap-south-1 --image-tag lastest_Version_Build."${BUILD_NUMBER}" --image-manifest "$MANIFEST"
                                    '''
                                    sh 'exit 1'
                                }
                            }
                        }

                        stage('Build and Push to PREPROD ECR') {
                            steps {
                                load "./jenkins/PREPROD/env.jenkins"
                                sh '/usr/local/bin/aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ${AWS_ID}.dkr.ecr.ap-south-1.amazonaws.com'
                                sh 'docker build --build-arg PORT=${PORT} -t ${image} .'
                                sh 'docker tag ${image}:latest ${AWS_ID}.dkr.ecr.ap-south-1.amazonaws.com/${image}:latest'
                                sh 'docker push ${AWS_ID}.dkr.ecr.ap-south-1.amazonaws.com/${image}:latest'
                            }
                        }

                        stage('Deploy into PREPROD ECS service') {
                            steps {
                                load "./jenkins/PREPROD/env.jenkins"
                                sh '/usr/local/bin/aws ecs update-service --force-new-deployment --service ${service} --cluster ${cluster}'
                            }
                        }
                    }
                }

                stage("PROD") {
                    when { expression { params.DEPLOY_TO == "PROD" } }
                    stages {
                        stage('Approval for prod') {
    steps {
        script {
            withCredentials([
                string(credentialsId: 'PRODUCT_OWNER_EMAIL', variable: 'productOwnerEmail'),
                string(credentialsId: 'TO_EMAIL', variable: 'toEmail'),
                string(credentialsId: 'OWNER_JENKIN_USER', variable: 'ownerJenkinsUser')
            ]) {
                echo "Approval process initiated."
                def jobName = currentBuild.fullDisplayName
                
                emailext body: """Please go to console output of <a href="${BUILD_URL}input">Visit approval Console</a> to approve or Reject.<br>""", 
                mimeType: 'text/html',
                subject: "[Jenkins] ${jobName} Build Approval Request",
                from: "${toEmail}",
                to: "${productOwnerEmail}",
                recipientProviders: [[$class: 'CulpritsRecipientProvider']]
                
                try {
                    userInput = input submitter: "${ownerJenkinsUser}", message: 'Do you approve?'
                    echo "Approval received from ${ownerJenkinsUser}."
                } catch (org.jenkinsci.plugins.workflow.steps.FlowInterruptedException e) {
                    cause = e.causes.get(0)
                    echo "Aborted by ${cause.getUser().toString()}."
                    currentBuild.result = 'ABORTED'
                }
            }
        }
    }
}

                        stage('Update old PROD ECR version tag') {
                            steps {
                                load "./jenkins/PROD/env.jenkins"
                                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                    sh '''#!/bin/bash
                                    MANIFEST=$(/usr/local/bin/aws ecr batch-get-image --repository-name "${image}" --region ap-south-1 --image-ids imageTag=latest --query 'images[].imageManifest' --output text)
                                    /usr/local/bin/aws ecr put-image --repository-name "${image}" --region ap-south-1 --image-tag lastest_Version_Build."${BUILD_NUMBER}" --image-manifest "$MANIFEST"
                                    '''
                                    sh 'exit 1'
                                }
                            }
                        }

                        stage('Build and Push image to PROD ECR') {
                            steps {
                                load "./jenkins/PROD/env.jenkins"
                                sh '/usr/local/bin/aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ${AWS_ID}.dkr.ecr.ap-south-1.amazonaws.com'
                                sh 'docker build --build-arg PORT=${PORT} -t ${image} .'
                                sh 'docker tag ${image}:latest ${AWS_ID}.dkr.ecr.ap-south-1.amazonaws.com/${image}:latest'
                                sh 'docker push ${AWS_ID}.dkr.ecr.ap-south-1.amazonaws.com/${image}:latest'
                            }
                        }

                        stage('Restart Prod service') {
                            steps {
                                load "./jenkins/PROD/env.jenkins"
                                sh '/usr/local/bin/aws ecs update-service --force-new-deployment --service ${service} --cluster ${cluster}'
                            }
                        }
                    }
                }
            }
        }
    }
}
