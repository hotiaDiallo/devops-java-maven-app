# Complete CI/CD pipeline with Jenkins and Terraform

![Image](/images/archi.png)

In this project we are going to see how to do a complete CI/CD pipeline on Jenkins

<br>

![Image](/images/ci-cd.png)

## Quick start : create jenkins as docker container

Since we will run docker in the Jenkins (which is a docker container), we will mount docker as a volume: 

    docker run -p 8080:8080 -p 50000:50000 -d -v jenkins_home:/var/jenkins_home -v
    /var/run/docker.sock:/var/run/docker.sock -v $(which docker):/usr/bin/docker jenkins/jenkins:lts

<br>

## Prerequisities
- Install [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) inside of our jenkins container : to be able to execute terraform commands inside the Jenkinsfile
- Create an ssh key pair and add it as credentials in Jenkins

    - Option 1 : create ssh key-pair from TF configuration 

    - Option 2 : create ssh key-pair from AWS Management console 
- Add Terraform configuration to provision the server 

checkout to this [project where I explain how to automate EC2 provisioning with Terraform](https://github.com/hotiaDiallo/terraform-playground/tree/deploy-to-ec2)

We wiil create for this project : 

- Our Own VPC with a Subnet 
- A Route table and Internet Gateway
- Security Group to configure firewall rules for the EC2 instance
- EC2 Instance
 
[checkout to the configuration files](https://github.com/hotiaDiallo/devops-java-maven-app/tree/sshagent-terraform/terraform)

## Create Jenkinsfile 

## Step 1 :  Build Application 

    stage("build jar") {
        steps {
            script {
                echo 'building application jar...'
                sh 'mvn package'
            }
        }
    }

## Step 2 : Build and push image to dockerhub

It's a best practice to create reusable code for task that we use on every process : building and pushing docker images is almost the same regardless of the the application. 

This is why I create a [shared](https://github.com/hotiaDiallo/jenkins-shared-library) library for this step. 

Here is a configuration for using the shared library 

    library identifier: 'jenkins-shared-library@main', retriever: modernSCM(
        [$class: 'GitSCMSource',
         remote: 'https://github.com/hotiaDiallo/jenkins-shared-library.git',
         credentialsId: 'github-repo'
        ]
    )

Build and push image to dockerhub

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

## Step 3 : Provision EC2 server

    stage("Provision"){
        environment {
            AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
            AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
            TF_VAR_env_prefix = 'test'
        }
        steps {
            script {
                dir('terraform') {
                    sh "terraform init"
                    sh "terraform apply --auto-approve"
                    EC2_PUBLIC_IP = sh(
                        script: "terraform output ec2_public_ip",
                        returnStdout: true
                    ).trim()
                }
            }
        }
    }
As you see, the first step is to configure credentials for Terraform to talk to AWS. 

When the EC2 server is created, we will install on docker and docker compose on it. FOr that, we will use the `user-data` option. 

    user_data = file("entry-script.sh")

The entry-script.sh file content 

    #!/bin/bash
    sudo yum update -y && sudo yum install -y docker
    sudo systemctl start docker 
    sudo usermod -aG docker ec2-user

    # install docker-compose 
    sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose

## Step 3 : Deploy the application on EC2 server using docker compose

For that, we need to : 

- Add dockerhub credentials : EC2 server need to authenticate 
- Copy docker compose to the EC2 instance 
- Copy our script to EC2 instance 

        #!/usr/bin/env bash
        export IMAGE_NAME=$1
        export DOCKER_USER=$2
        export DOCKER_PWD=$3
        echo $DOCKER_PWD | docker login -u $DOCKER_USER --password-stdin
        docker-compose -f docker-compose.yaml up --detach
        echo "deploy succes"

- Execute the script commande with required parameters : EC2_PUBLIC_IP, Dockerhub credentials (DOCKER_CREDS_USR, DOCKER_CREDS_PSW)


        stage("deploy") {
            environment{
                DOCKER_CREDS = credentials('docker-hub-repo')
                // This gives us to env vars DOCKER_CREDS_USR and DOCKER_CREDS_PSW
            }
            steps {
                script {
                    echo "waiting for EC2 server to initialize" 
                    sleep(time: 90, unit: "SECONDS") 

                    echo 'deploying docker image to EC2...'
                    echo "${EC2_PUBLIC_IP}"

                    //def dockerCmd = "docker run -d -p 8080:8080 ${IMAGE_NAME}"
                    def shellCmd = "bash ./server-cmds.sh ${IMAGE_NAME} ${DOCKER_CREDS_USR} ${DOCKER_CREDS_PSW}"
                    def ec2InstanceServer = "ec2-user@${EC2_PUBLIC_IP}"
                    sshagent(['ec2-server-key']){
                        sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2InstanceServer}:/home/ec2-user"
                        sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2InstanceServer}:/home/ec2-user"
                        sh "ssh -o StrictHostKeyChecking=no ${ec2InstanceServer} ${shellCmd}"
                    }
                }
            }
        }

## Results

![Image](/images/ec2.png) 

- Logs for provision 

![Image](/images/provision.png)

- Logs for deploy 

![Image](/images/deploy.png)

- ssh to EC2 instance and verify application is running 

![Image](/images/docker-container.png)
