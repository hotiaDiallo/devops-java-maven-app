# Build Automation & CI/CD with Jenkins and AWS

![Image](/images/Jenkins-pipeline.drawio.png)

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

![Image](/images/step1.drawio.png)

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

![Image](/images/step2.drawio.png)

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

![Image](/images/step3.drawio.png)

In this step, Jenkins is going to need to authenticate to docker hub to push the build image; 
So it's we need to add credentials(`docker-hub-repo`) on jenkins before doing a `docker login`

Now we can use theses credentials in our pipeline and push docker image after the build 

Here is everything you need to build and push your image to `ECR`


![Image](/images/build-push-ecr.png)


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

## Deploy EKS 

![Image](/images/step4.drawio.png)

In this project we only have one image to deploy, but if we have more than one, it's will not be easy de deploy without using a container orchestrator. 

### [See how to create EKS cluster](https://github.com/hotiaDiallo/devops-java-maven-app/tree/eks-cluster-with-node-group)

Steps to deploy Image from Dockerhub to EKS

- Install [kubectl](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/) inside Jenkins container 
- Install [aws-iam-authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html) inside Jenkins container. 
Amazon EKS uses IAM to provide authentication to your Kubernetes cluster through the AWS IAM authenticator for Kubernetes
Follow this [link](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html) to see more more details on how to configure `aws-iam-authenticator`
- create a [kubeconfig](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html) file for the cluster : The kubectl command-line tool uses configuration information in kubeconfig files to communicate with the API server of a cluster

```
aws eks update-kubeconfig --region region-code --name my-cluster
```

In this project, we are going to use `eksctl` which uses `CloudFormation` to create our cluster

Follow this [Link](https://github.com/weaveworks/eksctl) to install `eksctl`


```
    eksctl create cluster \
    > --name devops-java-maven-app \
    > --region eu-west-3 \
    > --nodegroup-name devops-java-maven \
    > --node-type t2.micro \
    > --nodes 2 \
    > --nodes-min 1 \
    > --nodes-max 3

```

If you want to use Terraform, [Check this project where I explain how to create an EKS cluster](https://github.com/hotiaDiallo/terraform-playground/tree/provision-eks)

![Image](/images/eks-devops-app.png)

Create a Secret for Kubernetes to be able tu pull images from ECR 

```
    kubectl create secret docker-registry my-aws-registry-key \
    > --docker-server=761900873497.dkr.ecr.eu-west-3.amazonaws.com \
    > --docker-username=AWS \
    > --docker-password=[PASSWORD]
```

[PASSWORD] --> `aws ecr get-login-password --region eu-west-3`

Add aws credentials on Jenkins for authentication.

![Image](/images/jenkins-credentials.png)

Add Kubeconfig file on Jenkins: since we are using Jenkins as a docker container, we are going to create a `config` file in our server and then copy that config file to the jenkins user directory: `/var/jenkins_home/.kube/`

![Image](/images/config-for-k8s.png)

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

## - Commit updated version

![Image](/images/step5.drawio.png)

```
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

                sh "git remote set-url origin https://${USERNAME}:${PASSWORD}@github.com/hotiaDiallo/devops-java-maven-app.git"
                sh 'git add .'
                sh 'git commit -m "ci: version bump"'
                sh 'git push origin HEAD:jenkins-jobs'
            }
        }
    }
}
```
