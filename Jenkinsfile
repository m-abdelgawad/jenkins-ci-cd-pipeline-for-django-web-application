pipeline {

    agent any

    environment {
        commit_id = ''
        automagic_app_image = ''
        automagic_db_image = ''
        server_monitor_image = ''
        recipient_name = ''
        commit_message = ''
    }

    stages {

        stage('Preparation') {
            steps {
                // error("Error occurred: This is a test exception!") // Exception test
                cleanWs()
                checkout scm
                script {
                    commit_id = sh(returnStdout: true, script: "git rev-parse --short HEAD").trim()
                    recipient_name = "Mohamed"
                    commit_message = sh(returnStdout: true, script: 'git log -1 --pretty=format:"%s"').trim()
                    automagic_app_image = "mabdelgawad94/automagic_developer:app-${commit_id}"
                    automagic_db_image = "mabdelgawad94/automagic_developer:db-${commit_id}"
                    server_monitor_image = "mabdelgawad94/automagic_developer:server_monitor-${commit_id}"
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
                            cd ./automagic-app/
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

                        docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {

                            def appImage = docker.build(automagic_app_image, './automagic-app').push()

                            def dbImage = docker.build(automagic_db_image, './automagic-db').push()

                            def serverMonitorImage = docker.build(server_monitor_image, './server-monitor').push()
                        }

                        // Clean up
                        sh "docker image prune -af"

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

                            echo "Automagic App Image: ${automagic_app_image}"
                            echo "Automagic DB Image: ${automagic_db_image}"
                            echo "Server Monitor Image: ${server_monitor_image}"

                            apt update
                            apt install -y sshpass

                            sshpass -p "${PASSWORD}" ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${USERNAME}@${PROD_IP} << EOF

                                # Export images names as environment variables
                                export AUTOMAGIC_APP_IMAGE=${automagic_app_image}
                                export AUTOMAGIC_DB_IMAGE=${automagic_db_image}
                                export SERVER_MONITOR_IMAGE=${server_monitor_image}

                                sudo apt update
                                sudo apt install -y p7zip-full

                                cd ./automagic-stack/
                                git pull
                                7z x -o./ -aoa -p${ENV_PASSWORD} ./env.7z
                                docker-compose -f pipeline-docker-compose.yml up --build -d

                                docker image prune -af
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
                emailext subject: "Jenkins Pipeline (AutoMagic Developer Website) - Success",
                         body: """
                               Dear ${recipient_name},<br>
                               <br>
                               The Jenkins pipeline has completed successfully.<br>
                               <br>
                               Commit ID: ${commit_id}<br>
                               Commit Message: ${commit_message}<br>
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
                emailext subject: "Jenkins Pipeline (AutoMagic Developer Website) - Failure",
                         body: """
                               Dear ${recipient_name},<br>
                               <br>
                               The Jenkins pipeline has failed.<br>
                               <br>
                               Commit ID: ${commit_id}<br>
                               Commit Message: ${commit_message}<br>
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