pipeline {
    agent none
    stages {
        stage('worker-build') {
            when {
                changeset '**/worker/**'
            }
            agent {
                docker {
                    image 'maven:3.9.6-sapmachine-17'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                echo 'Compiling worker app..'
                dir('worker') {
                    sh 'mvn compile'
                }
            }
        }
        stage('vote-build') {
            when {
                changeset '**/vote/**'
            }
            agent {
                docker {
                    image 'python:2.7-alpine'
                    args '-u root --privileged'
                }
            }
            steps {
                echo 'Compiling vote app..'
                dir('vote') {
                    sh 'pip install -r requirements.txt'
                }
            }
        }
        stage('result-build') {
            when {
                changeset '**/result/**'
            }
            agent {
                docker {
                    image 'node:8.16.0-alpine'
                }
            }
            steps {
                echo 'Compiling result app..'
                dir('result') {
                    sh 'npm install'
                }
            }
        }
        stage('worker-test') {
            when {
                changeset '**/worker/**'
            }
            agent {
                docker {
                    image 'maven:3.9.6-sapmachine-17'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                echo 'Running Unit Tests on worker app..'
                dir('worker') {
                    sh 'mvn clean test'
                }
            }
        }
        stage('vote-test') {
            agent {
                docker {
                    image 'python:2.7-alpine'
                    args '-u root --privileged'
                }
            }
            when {
                changeset '**/vote/**'
            }
            steps {
                echo 'Running Unit Tests on vote app.'
                dir(path: 'vote') {
                    sh 'pip install -r requirements.txt'
                    sh 'nosetests -v'
                }
            }
        }
        stage('vote-integration') {
            agent any
            when {
                changeset '**/vote/**'
                branch 'master'
            }
            steps {
                echo 'Running Integration Tests on vote app'
                dir('vote') {
                    sh 'sh integration_test.sh'
                }
            }
        }
        stage('result-test') {
            when {
                changeset '**/result/**'
            }
            agent {
                docker {
                    image 'node:8.16.0-alpine'
                }
            }
            steps {
                echo 'Running Unit Tests on result app..'
                dir('result') {
                    sh 'npm install'
                    sh 'npm test'
                }
            }
        }
        stage('worker-package') {
            when {
                branch 'master'
                changeset '**/worker/**'
            }
            agent {
                docker {
                    image 'maven:3.9.6-sapmachine-17'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                echo 'Packaging worker app'
                dir('worker') {
                    sh 'mvn package -DskipTests'
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                }
            }
        }
        stage('vote integration') {
            agent any
            when {
                changeset '**/vote/**'
                branch 'master'
            }
            steps {
                echo 'Running Integration Tests on vote app'
                dir('vote') {
                    sh 'sh integration_test.sh'
                }
            }
        }
        stage('worker-docker-package') {
            agent any
            when {
                branch 'master'
                changeset '**/worker/**'
            }
            steps {
                echo 'Packaging worker app with docker'
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerlogin') {
                        def workerImage = docker.build("kzwolenik95/worker:v${env.BUILD_ID}", './worker')
                        workerImage.push()
                        workerImage.push("${env.BRANCH_NAME}")
                        workerImage.push('latest')
                    }
                }
            }
        }
        stage('vote-docker-package') {
            agent any
            when {
                branch 'master'
                changeset '**/vote/**'
            }
            steps {
                echo 'Packaging vote app with docker'
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerlogin') {
                        def voteImage = docker.build("kzwolenik95/vote:v${env.BUILD_ID}", './vote')
                        voteImage.push()
                        voteImage.push("${env.BRANCH_NAME}")
                        voteImage.push('latest')
                    }
                }
            }
        }
        stage('result-docker-package') {
            agent any
            when {
                branch 'master'
                changeset '**/result/**'
            }
            steps {
                echo 'Packaging result app with docker'
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerlogin') {
                        def resultImage = docker.build("kzwolenik95/result:v${env.BUILD_ID}", './result')
                        resultImage.push()
                        resultImage.push("${env.BRANCH_NAME}")
                        resultImage.push('latest')
                    }
                }
            }
        }
        stage('Sonarqube') {
            agent any
            when {
                branch 'master'
            }
            environment {
                sonarpath = tool 'SonarScanner'
            }
            steps {
                echo 'Running Sonarqube Analysis..'
                withSonarQubeEnv('sonar-instavote') {
                    sh "${sonarpath}/bin/sonar-scanner -Dproject.settings=sonar-project.properties -Dorg.jenkinsci.plugins.durabletask.BourneShellScript.HEARTBEAT_CHECK_INTERVAL=86400"
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                    // true = set pipeline to UNSTABLE, false = don't
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('deploy to dev') {
            agent any
            when {
                branch 'master'
            }
            steps {
                echo 'Deploy instavote app with docker compose'
                sh 'docker-compose up -d'
            }
        }
    }
    post {
        always {
            slackSend(channel: '#ci-cd', message: "Build completed: ${currentBuild.currentResult} ${env.JOB_NAME} ${env.BUILD_NUMBER}")
            echo 'Building mono pipeline for voting app is completed..'
        }
    }
}
