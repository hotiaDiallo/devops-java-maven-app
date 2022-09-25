# Build Automation & CI/CD with Jenkins and AWS

![Image](/images/full-blue-ocean-pipeline.png)

### Quick start : create jenkins container with mounted docker

```
docker run -p 8080:8080 -p 50000:50000 -d -v jenkins_home:/var/jenkins_home -v
/var/run/docker.sock:/var/run/docker.sock -v $(which docker):/usr/bin/docker jenkins/jenkins:lts
```

## Create a simple pipeline to build and push image on docker hub

[simple jenkinsfile pipeline](https://github.com/hotiaDiallo/devops-java-maven-app/blob/jenkins-jobs/Jenkinsfile-simple-pipeline/Jenkinsfile)

<br>

## Jenkins Shared Library 

1- Create shared Library project 

[Created Shared Library Project/Repository](https://github.com/hotiaDiallo/jenkins-shared-library)

2- Used Shared Library in Jenkinsﬁle
- Use Parameters in Shared Library
- Extract logic into Groovy Classes
- Deﬁne Shared Library in Jenkinsﬁle directly (project scoped)

[jenkinsfile with shared library](https://github.com/hotiaDiallo/devops-java-maven-app/tree/jenkins-jobs/jenkins-shared-library)

## Jenkins Pipeline to deploy on AWS EC2

Deploy Java Maven App via Jenkins Pipeline on EC2 Instance
using Docker-Compose File

## Prerequisite
- Install docker compose on the EC2 server
```
sudo curl -L "https://github.com/docker/compose/releases/download/2.11.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
check the version : https://github.com/docker/compose/releases 

- Next, set the permissions to make the binary executable

```
sudo chmod +x /usr/local/bin/docker-compose
```
- Then, verify that the installation was successful by checking the version

```
docker-compose --version
```

[jenkinsfile to deploy to EC2 server](https://github.com/hotiaDiallo/devops-java-maven-app/blob/jenkins-jobs/Jenkinsfile)

![Image](/images/deploy-with-docker-compose.png)
![Image](/images/full-pipeline.png)