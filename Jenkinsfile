/* This is when shared library is set for all projects*/
@Library('jenkins-shared-library')


/* This is when shared library is set only for this project*/
/* library identifier: 'jenkins-shared-library@master', retriever: modernSCM(
        [$class: 'GitSCMSource',
         remote: 'https://github.com/hotiaDiallo/jenkins-shared-library.git',
         credentialsId: 'github-credentials'
        ]
) */

def groovy


pipeline {
    agent any
    tools {
        maven 'maven'
    }
    stages {
        // in case we have a script specfic for this project
        stage("init") {
            steps {
                script {
                    groovy = load "script.groovy"
                }
            }
        }
        stage("build jar") {
            steps {
                script {
                    groovy.buildJar()
                }
            }
        }
        stage("build and push image") {
            steps {
                script {
                    dockerBuildImage 'selftaughdevops/java-maven-app:1.0'
                    dockerLogin()
                    dockerPush 'selftaughdevops/java-maven-app:1.0'
                }
            }
        }
        stage("deploy") {
            steps {
                script {
                    groovy.deployApp()
                }
            }
        }
    }
}