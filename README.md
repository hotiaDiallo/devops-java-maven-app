# Build Automation & CI/CD with Jenkins and AWS

In this project we are going to see how to do a complete CI/CD pipeline on Jenkins

## Quick start : create jenkins as docker container

Since we will run docker in the Jenkins (which is a docker container), we will mount docker as a volume: 

    docker run -p 8080:8080 -p 50000:50000 -d -v jenkins_home:/var/jenkins_home -v
    /var/run/docker.sock:/var/run/docker.sock -v $(which docker):/usr/bin/docker jenkins/jenkins:lts`

## Create a simple pipeline to build and push image on docker hub fo testing

[simple jenkinsfile pipeline](https://github.com/hotiaDiallo/devops-java-maven-app/blob/jenkins-jobs/Jenkinsfile-simple-pipeline/Jenkinsfile)

## Our CI/CD steps 
- Update application version
- Build the application 
- Build docker image and push to dockerhub or private repository
- Deploy to AWS/LKE
- Commit the update version 

### Update application version

Since it's a java project, we are going to use a [maven plugin](https://www.mojohaus.org/build-helper-maven-plugin/parse-version-mojo.html) update the version every time we run the pipeline. 

So we have to install maven on Jenkins. 

    tools {
        maven 'maven'
    }

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

<br>

### Build the application
```
stage('build app') {
    steps {
        script {
            echo "building the application..."
            sh 'mvn clean package'
        }
    }
}
```
<br>

### Build docker image and push to dockerhub or private repository

In this step, Jenkins is going to need to authenticate to docker hub to push the build image; 
So it's we need to add credentials(`docker-hub-repo`) on jenkins before doing a `docker login`

Now we can use theses credentials in our pipeline and push docker image after the build 

```
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
```
<br>

## Deploy to AWS : EC2 and EKS 

In this project we only have one image to deploy, but if we have more than one, it's will not be easy de deploy without using a container orchestrator. 

## - Use docker compose to deploy to AWS EC2 instance

For that ssh to the EC2 server : 

 - Install docker compose on the EC2 server
```
sudo curl -L "https://github.com/docker/compose/releases/download/2.11.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
check the version : https://github.com/docker/compose/releases 

- Next, set the permissions to make the file executable

```
sudo chmod +x /usr/local/bin/docker-compose
```
- Then, verify that the installation was successful by checking the version

```
docker-compose --version
```
- install [sshagent](https://www.jenkins.io/doc/pipeline/steps/ssh-agent/) plugin ang update the deploy step

```
    stage("deploy") {
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
    }
```

## - Deploy to EKS

Steps to deploy Image from Dockerhub to EKS

- Install [kubectl](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/) inside Jenkins container 
- Install [aws-iam-authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html) inside Jenkins container. 
Amazon EKS uses IAM to provide authentication to your Kubernetes cluster through the AWS IAM authenticator for Kubernetes
- create a [kubeconfig](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html) file for the cluster : The kubectl command-line tool uses configuration information in kubeconfig files to communicate with the API server of a cluster

```
aws eks update-kubeconfig --region region-code --name my-cluster
```

Then update the deploy step on Jenkinsfile

```
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
```
<br>

---
## Jenkins Shared Library 

1- Create shared Library project 

[Created Shared Library Project/Repository](https://github.com/hotiaDiallo/jenkins-shared-library)

2- Used Shared Library in Jenkinsﬁle
- Use Parameters in Shared Library
- Extract logic into Groovy Classes
- Deﬁne Shared Library in Jenkinsﬁle directly (project scoped)

[jenkinsfile with shared library](https://github.com/hotiaDiallo/devops-java-maven-app/tree/jenkins-jobs/jenkins-shared-library)
