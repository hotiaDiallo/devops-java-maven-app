/* This is when shared library is set for all projects*/
@Library('jenkins-shared-library')_


/* This is when shared library is set only for this project*/
/* library identifier: 'jenkins-shared-library@main', retriever: modernSCM(
        [$class: 'GitSCMSource',
         remote: 'https://github.com/hotiaDiallo/jenkins-shared-library.git',
         credentialsId: 'github-repo'
        ]
) */

pipeline {
    agent any
    tools {
        maven 'maven'
    }
    environment{
        IMAGE_NAME = 'selftaughdevops/java-maven-app:2.0.0'
    }
    stages {
        // in case we have a script specfic for this project
    
        stage("build jar") {
            steps {
                script {
                    echo 'building application jar...'
                    sh 'mvn package'
                }
            }
        }
        stage("build and push image") {
            steps {
                script {
                    echo 'building image and pushing to dockerhub'
                    dockerBuildImage(env.IMAGE_NAME)
                    dockerLogin()
                    dockerPush(env.IMAGE_NAME)
                }
            }
        }

        stage("Provision"){
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
                TF_VAR_env_prefix = 'test'
            }
            steps{
                script{
                     echo 'Provisionning EC2 server...'
                     dir('terraform'){
                        sh 'terraform init'
                        sh 'terraform apply --auto-approve'
                        // get ec2 public ip value
                        EC2_PUBLIC_IP = sh(
                            script: 'terraform output ec2_public_ip'
                            returnStdout: true
                        ).trim()
                     }
                }
            }
        }

        stage("deploy") {
            steps {
                script {
                    echo "waiting for EC2 server to initialize" 
                    sleep(time: 90, unit: "SECONDS") 

                    echo 'deploying docker image to EC2...'
                    echo "${EC2_PUBLIC_IP}"

                    //def dockerCmd = "docker run -d -p 8080:8080 ${IMAGE_NAME}"
                    def shellCmd = "bash ./server-cmds.sh ${IMAGE_NAME}"
                    def ec2InstanceServer = "ec2-user@${EC2_PUBLIC_IP}"
                    sshagent(['ec2-server-key']){
                        sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2InstanceServer}:/home/ec2-user"
                        sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2InstanceServer}:/home/ec2-user"
                        sh "ssh -o StrictHostKeyChecking=no ${ec2InstanceServer} ${shellCmd}"
                    }
                }
            }
        }
    }
}