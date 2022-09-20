pipeline {
    agent any
    tools{
        maven 'maven'
    }
    stages {
        stage("build jar") {
            steps {
                script {
                    echo "building jar..."
                    sh 'mvn package'
                }
            }
        }
        stage("build image") {
            steps {
                script {
                    echo "building the docker image..."
                    withCredentials([usernamePassword(
                        credentialsId:'docker-hub-repo', 
                        usernameVariable: 'USERNAME', 
                        passwordVariable: 'PASSWORD'
                    )]){
                        sh 'docker build -t selftaughdevops/devops-java-maven-app:1.0 .'
                        sh "echo $PASSWORD | docker login -u $USERNAME --password-stdin"
                        sh 'docker push selftaughdevops/devops-java-maven-app:1.0'
                    }
                }
            }
        }
        stage("deploy") {
            steps {
                script {
                    echo "deploying"
                }
            }
        }
    }   
}