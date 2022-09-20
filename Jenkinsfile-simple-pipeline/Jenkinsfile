pipeline {
    agent any
    tools{
        maven 'maven'
    }
    parameters{
        choice(name:'VERSION', choices:['1.0.0', '1.1.0', '1.2.0'], description: '')
        booleanParam(name:'executeTests', defaultValue: true, description:'')
    }
    stages {

        stage("tests") {
            when {
                expression {
                    params.executeTests
                }
            }
            steps {
                script {
                    echo "Launching tests..."
                }
            }
        }

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
                    env.ENV = input message: "Select the environment to deploy to", ok: "Done", parameters: [choice(name: 'ONE', choices: ['dev', 'staging', 'prod'], description: '')]
                    echo "Deploying to ${ENV}"
                }
            }
        }
    }   
}