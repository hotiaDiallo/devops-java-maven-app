#!/usr/bin/env groovy

def groovy

pipeline {
    agent any
    tools{
        maven 'maven'
    }
    stages {

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
        stage("build image") {
            steps {
                script {
                   groovy.buildImage()
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