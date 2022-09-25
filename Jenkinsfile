/* This is when shared library is set for all projects*/
@Library('jenkins-shared-library')_


/* This is when shared library is set only for this project*/
/*library identifier: 'jenkins-shared-library@main', retriever: modernSCM(
        [$class: 'GitSCMSource',
         remote: 'https://github.com/hotiaDiallo/jenkins-shared-library.git',
         credentialsId: 'github-credentials'
        ]
)*/

pipeline {
    agent any
    tools {
        maven 'maven'
    }
    environment{
        IMAGE_NAME = 'selftaughdevops/java-maven-app:1.0-SNAPSHOT'
    }
    stages {

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
        
        stage('build app') {
            steps {
                script {
                    echo "building the application..."
                    sh 'mvn clean package'
                }
            }
        }
        stage("build and push image") {
            steps {
                script {
                    dockerBuildImage(env.IMAGE_NAME)
                    dockerLogin()
                    dockerPush(env.IMAGE_NAME)
                }
            }
        }
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

                        //sh "git remote set-url origin https://${USERNAME}:${PASSWORD}@github.com/hotiaDiallo/devops-java-maven-app.git"
                        //sh 'git add .'
                        //sh 'git commit -m "ci: version bump"'
                        //sh 'git push origin HEAD:jenkins-jobs'
                    }
                }
            }
        }
    }
}
