pipeline {
    agent none

    triggers {
        pollSCM 'H/10 * * * *'
        upstream(upstreamProjects: "spring-data-commons/master", threshold: hudson.model.Result.SUCCESS)
    }

    options {
        disableConcurrentBuilds()
    }

    stages {
        stage("Docker images") {
            parallel {
                stage('Publish JDK 8 + MongoDB 4.0') {
                    when {
                        changeset "ci/openjdk8-mongodb-4.0/**"
                    }
                    agent any

                    steps {
                        script {
                            def image = docker.build("springci/spring-data-openjdk8-with-mongodb-4.0", "ci/openjdk8-mongodb-4.0/")
                            docker.withRegistry('', 'hub.docker.com-springbuildmaster') {
                                image.push()
                            }
                        }
                    }
                }
                stage('Publish JDK 8 + MongoDB 4.1') {
                    when {
                        changeset "ci/openjdk8-mongodb-4.1/**"
                    }
                    agent any

                    steps {
                        script {
                            def image = docker.build("springci/spring-data-openjdk8-with-mongodb-4.1", "ci/openjdk8-mongodb-4.1/")
                            docker.withRegistry('', 'hub.docker.com-springbuildmaster') {
                                image.push()
                            }
                        }
                    }
                }
            }
        }

        stage("test: baseline") {
            agent {
                docker {
                    image 'springci/spring-data-openjdk8-with-mongodb-4.0:latest'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                sh 'mkdir -p /tmp/mongodb/db /tmp/mongodb/log'
                sh 'mongod --dbpath /tmp/mongodb/db --replSet rs0 --fork --logpath /tmp/mongodb/log/mongod.log &'
                sh 'sleep 10'
                sh 'mongo --eval "rs.initiate({_id: \'rs0\', members:[{_id: 0, host: \'127.0.0.1:27017\'}]});"'
                sh 'sleep 15'
                sh './mvnw clean dependency:list test -Dsort -B'
            }
        }

        stage("Test other configurations") {
            parallel {
                stage("test: mongodb 4.1") {
                    agent {
                        docker {
                            label 'data'
                            image 'springci/spring-data-openjdk8-with-mongodb-4.1:latest'
                            args '-v $HOME/.m2:/root/.m2'
                        }
                    }
                    steps {
                        sh 'mkdir -p /tmp/mongodb/db /tmp/mongodb/log'
                        sh 'mongod --dbpath /tmp/mongodb/db --replSet rs0 --fork --logpath /tmp/mongodb/log/mongod.log &'
                        sh 'sleep 10'
                        sh 'mongo --eval "rs.initiate({_id: \'rs0\', members:[{_id: 0, host: \'127.0.0.1:27017\'}]});"'
                        sh 'sleep 15'
                        sh './mvnw clean dependency:list test -Dsort -B'
                    }
                }
            }
        }

        stage('Release to artifactory') {
            when {
                branch 'issue/*'
            }
            agent {
                docker {
                    image 'adoptopenjdk/openjdk8:latest'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }

            environment {
                ARTIFACTORY = credentials('02bd1690-b54f-4c9f-819d-a77cb7a9822c')
            }

            steps {
                sh "./mvnw -Pci,snapshot -Dmaven.test.skip=true clean deploy -B"
            }
        }

        stage('Release to artifactory with docs') {
            when {
                branch 'master'
            }
            agent {
                docker {
                    image 'adoptopenjdk/openjdk8:latest'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }

            environment {
                ARTIFACTORY = credentials('02bd1690-b54f-4c9f-819d-a77cb7a9822c')
            }

            steps {
                sh "./mvnw -Pci,snapshot -Dmaven.test.skip=true clean deploy -B"
            }
        }
    }

    post {
        changed {
            script {
                slackSend(
                        color: (currentBuild.currentResult == 'SUCCESS') ? 'good' : 'danger',
                        channel: '#spring-data-dev',
                        message: "${currentBuild.fullDisplayName} - `${currentBuild.currentResult}`\n${env.BUILD_URL}")
                emailext(
                        subject: "[${currentBuild.fullDisplayName}] ${currentBuild.currentResult}",
                        mimeType: 'text/html',
                        recipientProviders: [[$class: 'CulpritsRecipientProvider'], [$class: 'RequesterRecipientProvider']],
                        body: "<a href=\"${env.BUILD_URL}\">${currentBuild.fullDisplayName} is reported as ${currentBuild.currentResult}</a>")
            }
        }
    }
}
