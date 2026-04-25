def gv

pipeline {
    agent any

    tools {
        maven 'maven-3.9'
    }

    stages {

        stage('increment version') {
            steps {
                script {
                    echo 'incrementing app version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\${parsedVersion.majorVersion}.\\${parsedVersion.minorVersion}.\\${parsedVersion.nextIncrementalVersion} \
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
                    echo 'building the application...'
                    sh 'mvn clean package'
                }
            }
        }

        stage('build image') {
            steps {
                script {
                    echo 'building the docker image...'
                    withCredentials([usernamePassword(credentialsId: 'dockerhub_credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "docker build -t roczyno/demo-app:${IMAGE_NAME} ."
                        sh 'echo $PASS | docker login -u $USER --password-stdin'
                        sh "docker push roczyno/demo-app:${IMAGE_NAME}"
                    }
                }
            }
        }

        stage('deploy') {
            steps {
                script {
                    echo 'deploying docker image...'
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

                            git status
                            git branch

                            git add .
                            git commit -m "ci: version bump" || echo "No changes to commit"

                            git push https://$USER:$PASS@github.com/roczyno/jenkins_java_maven_app.git HEAD:jenkins-jobs
                        '''
                    }
                }
            }
        }

    }
}
