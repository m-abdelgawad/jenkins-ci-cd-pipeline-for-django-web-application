pipeline {

    agent any

    environment {
        commit_id = ''
    }

    stages {

        stage('Preparation') {
            steps {
                // error("Error occurred: This is a test exception!") // Exception test
                cleanWs()
                checkout scm
                script {
                    commit_id = sh(returnStdout: true, script: "git rev-parse --short HEAD").trim()
                }
            }
        }

        stage('Testing The Django Web Application') {
            steps {
                script {

                    def testContainer = docker.image('python:3.10')
                    testContainer.pull()

                    testContainer.inside("-u root") {

                        sh '''
                            cd ./automagic-website/
                            pip3 install -r req.txt
                            python3 manage.py test
                        '''

                    }  // End container block

                }  // End script block
            }  // End steps block
        }  // End testing stage block

        stage('Docker Build & Publish') {
            steps {
                script {

                    def buildContainer = docker.image('ubuntu:latest')
                    buildContainer.pull()

                    buildContainer.inside("-u root") {

                        sh '''
                            apt update
                            apt install -y apt-transport-https ca-certificates curl software-properties-common
                            curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
                            add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
                            apt-cache policy docker-ce
                            apt install -y docker-ce
                        '''

                        def appImageName = "mabdelgawad94/automagic_developer:app-${commit_id}"
                        def dbImageName = "mabdelgawad94/automagic_developer:db-${commit_id}"
                        def serverMonitorImageName = "mabdelgawad94/automagic_developer:server_monitor-${commit_id}"

                        buildContainer.withRun("-u root") {
                            docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {

                                def appImage = docker.build(appImageName, './automagic-website').push()

                                def dbImage = docker.build(dbImageName, './automagic-db').push()

                                def serverMonitorImage = docker.build(serverMonitorImageName, './server-monitor').push()
                            }
                        }

                        // Clean up
                        sh "docker rmi ${appImageName}"
                        sh "docker rmi ${dbImageName}"
                        sh "docker rmi ${serverMonitorImageName}"

                    }  // End container block
                }
            }
        }  // End Docker Stage

        stage('Deployment on Production Server') {
            steps {
                script {

                    def deployContainer = docker.image('ubuntu:latest')
                    deployContainer.pull()

                    deployContainer.inside("-u root") {
                        withCredentials([string(credentialsId: 'envPassword', variable: 'ENV_PASSWORD'), string(credentialsId: 'prodIP', variable: 'PROD_IP'), usernamePassword(credentialsId: 'prodUser', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh """
                            apt update
                            apt install -y sshpass p7zip-full
                            sshpass -p "${PASSWORD}" ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${USERNAME}@${PROD_IP} << EOF
                            cd ./automagic-stack/
                            7z x -o./ -aoa -p${ENV_PASSWORD} ./env.7z
                            git pull
                            docker-compose up --build -d
EOF
                        """
                        }
                    }  // End container block
                }
            }
        }  // End Deployment Stage

    }  // End stages block

    post {
        success {
            // Send success email notification
            script {
                def commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                def commitMessage = sh(returnStdout: true, script: 'git log -1 --pretty=format:"%s"').trim()
                def recipientName = "Mohamed"

                emailext subject: "Jenkins Pipeline (AutoMagic Developer Website) - Success",
                         body: """
                               Dear ${recipientName},<br>
                               <br>
                               The Jenkins pipeline has completed successfully.<br>
                               <br>
                               Commit ID: ${commitId}<br>
                               Commit Message: ${commitMessage}<br>
                               <br>
                               Regards,<br>
                               Jenkins
                               """,
                         to: "muhammadabdelgawwad@gmail.com",
                         mimeType: 'text/html',
                         recipientProviders: [[$class: 'CulpritsRecipientProvider'], [$class: 'RequesterRecipientProvider'], [$class: 'DevelopersRecipientProvider']]
            }
        }
        failure {
            // Send failure email notification with exception details
            script {
                def commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                def commitMessage = sh(returnStdout: true, script: 'git log -1 --pretty=format:"%s"').trim()
                def recipientName = "Mohamed"

                emailext subject: "Jenkins Pipeline (AutoMagic Developer Website) - Failure",
                         body: """
                               Dear ${recipientName},<br>
                               <br>
                               The Jenkins pipeline has failed.<br>
                               <br>
                               Commit ID: ${commitId}<br>
                               Commit Message: ${commitMessage}<br>
                               <br>
                               Regards,<br>
                               Jenkins
                               """,
                         to: "muhammadabdelgawwad@gmail.com",
                         mimeType: 'text/html',
                         recipientProviders: [[$class: 'CulpritsRecipientProvider'], [$class: 'RequesterRecipientProvider'], [$class: 'DevelopersRecipientProvider']]
            }
        }
    }  // End post

}  // End pipeline block