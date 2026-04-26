#!/usr/bin/env groovy
library identifier: 'jenkins-shared-library@master', retriever: modernSCM(
        [$class: 'GitSCMSource',
        remote: 'https://github.com/roczyno/jenkins_shared_library',
        credentialsId: 'github_credentials'])

def gv

pipeline {   
    agent any
    tools {
        maven 'maven-3.9'
    }

    environment {
        IMAGE_NAME = 'roczyno/demo-app:java-maven-1.0'
    }



    stages {
        stage("init") {
            steps {
                script {
                    gv = load "script.groovy"
                }
            }
        }

        stage("build jar") {
            steps {
                script {
                    buildJar()
                }
            }
        }

        stage("build and push image") {
            steps {
                script {
                    buildImage "${IMAGE_NAME}:jma-3.0"
                    dockerLogin()
                    dockerPush "${IMAGE_NAME}:jma-3.0"
                }
            }
        }
        
        stage("deploy") {
            steps {
                script {
                    echo "Deploying docker image to EC2"
                    def installDockerCMD = "sudo yum install -y docker && sudo systemctl start docker && sudo usermod -aG docker \$(whoami)"
                    def dockerRunCMD = "docker run -d -p 8080:8080 ${IMAGE_NAME}:jma-3.0"
                    withCredentials([usernamePassword(credentialsId: 'dockerhub_credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sshagent(['ec2-server-key']) {
                            sh "ssh -o StrictHostKeyChecking=no ec2-user@3.85.4.100 'which docker || (${installDockerCMD})'"
                            sh "ssh -o StrictHostKeyChecking=no ec2-user@3.85.4.100 'echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin'"
                            sh "ssh -o StrictHostKeyChecking=no ec2-user@3.85.4.100 '${dockerRunCMD}'"
                        }
                    }
                }
            }
        }               
    }
}
