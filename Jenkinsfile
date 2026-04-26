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
        IMAGE_REPO = 'roczyno/demo-app'
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
                    env.IMAGE_NAME = "${IMAGE_REPO}:${version}-${BUILD_NUMBER}"
                }
            }
        }

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
                    buildImage(env.IMAGE_NAME)
                    dockerLogin()
                    dockerPush(env.IMAGE_NAME)
                }
            }
        }
        
        stage("deploy") {
            steps {
                script {
                    echo "Deploying docker image to EC2"
                    def installDockerCMD = "sudo yum install -y docker && sudo systemctl start docker && sudo usermod -aG docker \$(whoami)"
                    def installComposeCMD = "sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-\$(uname -s)-\$(uname -m) -o /usr/local/bin/docker-compose && sudo chmod +x /usr/local/bin/docker-compose"
                    withCredentials([usernamePassword(credentialsId: 'dockerhub_credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sshagent(['ec2-server-key']) {
                            sh "ssh -o StrictHostKeyChecking=no ec2-user@3.85.4.100 'which docker || (${installDockerCMD})'"
                            sh "ssh -o StrictHostKeyChecking=no ec2-user@3.85.4.100 'which docker-compose || (${installComposeCMD})'"
                            sh "ssh -o StrictHostKeyChecking=no ec2-user@3.85.4.100 'echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin'"
                            sh "scp -o StrictHostKeyChecking=no docker-compose.yml ec2-user@3.85.4.100:/home/ec2-user/docker-compose.yml"
                            sh "ssh -o StrictHostKeyChecking=no ec2-user@3.85.4.100 'IMAGE=${env.IMAGE_NAME} docker-compose -f /home/ec2-user/docker-compose.yml up -d'"
                        }
                    }
                }
            }
        } 


        stage('commit version update') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github_credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh '''
                            git config --global user.email "jenkins@example.com"
                            git config --global user.name "jenkins"

                            git remote set-url origin https://$USER:$PASS@github.com/roczyno/jenkins_java_maven_app.git
                            git fetch origin jenkins-shared-lib
                            git checkout jenkins-shared-lib
                            git pull origin jenkins-shared-lib

                            git add pom.xml
                            git commit -m "ci: version bump" || echo "No changes to commit"

                            git push origin jenkins-shared-lib
                        '''
                    }
                }
            }
        }              
    }
}
