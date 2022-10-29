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
                    buildJar()
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

        stage("deploy") {
            steps {
                script {
                   echo 'deploying application to EC2...'
                }
            }
        }
    }
}