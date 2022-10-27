#!/usr/bin/env groovy

pipeline {
    agent any
    tools {
        maven 'maven'
    }
    stages {
        stage('increment version') {
            steps {
                script {
                    echo 'incrementing app version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                }
            }
        }
        stage('build app') {
            steps {
                script {
                    echo "building the application..."
                    sh 'mvn clean package'
                }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                    withCredentials([usernamePassword(
                            credentialsId: 'docker-hub-repo',
                            passwordVariable: 'PASSWORD',
                            usernameVariable: 'USERNAME'
                    )]) {
                        sh "docker build -t selftaughdevops/devops-java-maven-app:${IMAGE_NAME} ."
                        sh "echo $PASSWORD | docker login -u $USERNAME --password-stdin"
                        sh "docker push selftaughdevops/devops-java-maven-app:${IMAGE_NAME}"
                    }
                }
            }
        }
        // Deploy to EC2 using docker compose
        /* stage("deploy") {
            steps {
                script {
                    echo "Deploying application to the EC2 server..."
                    //def dockerCmd = "docker run -d -p 8080:8080 ${IMAGE_NAME}"
                    def shellCmd = "bash ./server-cmds.sh ${IMAGE_NAME}"
                    def ec2InstanceServer = 'ec2-user@44.211.190.125'
                    sshagent(['ec2-server-key']){
                        sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2InstanceServer}:/home/ec2-user"
                        sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2InstanceServer}:/home/ec2-user"
                        sh "ssh -o StrictHostKeyChecking=no ${ec2InstanceServer} ${shellCmd}"
                    }
                }
            }
        } */
        stage('deploy to eks cluster...') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
                APP_NAME = 'java-maven-app'
            }
            steps {
                script {
                    echo 'deploying docker image...'
                    sh 'envsubst < k8s/deployment.yaml | kubectl apply -f -'
                    sh 'envsubst < k8s/service.yaml | kubectl apply -f -'
                }
            }
        }
        stage('commit version update') {
            steps {
                script {
                    withCredentials([usernamePassword(
                            credentialsId: 'github-credentials',
                            passwordVariable: 'PASSWORD',
                            usernameVariable: 'USERNAME'
                    )]) {
                        // git config here for the first time run
                        sh 'git config --global user.email "idiallo@example.com"'
                        sh 'git config --global user.name "idiallo"'

                       /* sh "git remote set-url origin https://${USERNAME}:${PASSWORD}@github.com/hotiaDiallo/devops-java-maven-app.git"
                        sh 'git add .'
                        sh 'git commit -m "ci: version bump"'
                        sh 'git push origin HEAD:jenkins-jobs'*/
                    }
                }
            }
        }
    }
}
