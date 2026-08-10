pipeline {
    agent any

    tools {
    maven 'Maven-3.9.16'
}

    parameters {
        choice(
            name: 'TOMCAT_INSTANCE',
            choices: [
                'instance1',
                'instance2'
            ],
            description: 'Select Tomcat instance for deployment'
        )
    }

    environment {
        CATALINA_HOME = '/home/osboxes/tomcat/apache-tomcat-11.0.24'

        TOMCAT_HOST = '192.168.1.12'
        DEPLOY_USER = 'osboxes'

        APP_NAME = 'myapp'
        WAR_FILE = 'target/myapp.war'

        SSH_CREDENTIALS = 'tomcat-ssh-key'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh '''
                    echo "======================================"
                    echo "Building application"
                    echo "======================================"
                    mvn -version
                    mvn clean package -DskipTests
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    echo "======================================"
                    echo "Running tests"
                    echo "======================================"

                    mvn test
                '''
            }
        }

        stage('Prepare Deployment') {
            steps {
                script {

                    if (params.TOMCAT_INSTANCE == 'instance1') {

                        env.CATALINA_BASE =
                            '/home/osboxes/tomcat/tomcat-instance1'

                        env.STOP_SCRIPT =
                            "${env.CATALINA_HOME}/stop-instance1.sh"

                        env.START_SCRIPT =
                            "${env.CATALINA_HOME}/start-instance1.sh"

                        env.TOMCAT_PORT = '8080'

                    } else if (params.TOMCAT_INSTANCE == 'instance2') {

                        env.CATALINA_BASE =
                            '/home/osboxes/tomcat/tomcat-instance2'

                        env.STOP_SCRIPT =
                            "${env.CATALINA_HOME}/stop-instance2.sh"

                        env.START_SCRIPT =
                            "${env.CATALINA_HOME}/start-instance2.sh"

                        env.TOMCAT_PORT = '8081'

                    } else {
                        error "Invalid Tomcat instance"
                    }

                    echo "======================================"
                    echo "Deployment Configuration"
                    echo "======================================"
                    echo "Instance      : ${params.TOMCAT_INSTANCE}"
                    echo "CATALINA_HOME : ${env.CATALINA_HOME}"
                    echo "CATALINA_BASE : ${env.CATALINA_BASE}"
                    echo "Stop script   : ${env.STOP_SCRIPT}"
                    echo "Start script  : ${env.START_SCRIPT}"
                    echo "Tomcat port   : ${env.TOMCAT_PORT}"
                }
            }
        }

        stage('Deploy') {
            steps {
                sshagent(credentials: [env.SSH_CREDENTIALS]) {

                    sh '''
                        set -e

                        echo "======================================"
                        echo "Deploying to Tomcat"
                        echo "======================================"

                        echo "Stopping ${TOMCAT_INSTANCE}..."

                        ssh -o StrictHostKeyChecking=no \
                            ${DEPLOY_USER}@${TOMCAT_HOST} \
                            "${STOP_SCRIPT}"

                        echo "Tomcat stopped."

                        echo "Waiting for Tomcat to stop..."
                        sleep 5

                        echo "Removing old WAR..."

                        ssh -o StrictHostKeyChecking=no \
                            ${DEPLOY_USER}@${TOMCAT_HOST} \
                            "rm -f ${CATALINA_BASE}/webapps/${APP_NAME}.war"

                        echo "Removing exploded application..."

                        ssh -o StrictHostKeyChecking=no \
                            ${DEPLOY_USER}@${TOMCAT_HOST} \
                            "rm -rf ${CATALINA_BASE}/webapps/${APP_NAME}"

                        echo "Copying new WAR..."

                        scp -o StrictHostKeyChecking=no \
                            ${WAR_FILE} \
                            ${DEPLOY_USER}@${TOMCAT_HOST}:/tmp/${APP_NAME}.war

                        echo "Installing new WAR..."

                        ssh -o StrictHostKeyChecking=no \
                            ${DEPLOY_USER}@${TOMCAT_HOST} \
                            "mv /tmp/${APP_NAME}.war ${CATALINA_BASE}/webapps/${APP_NAME}.war"

                        echo "Starting ${TOMCAT_INSTANCE}..."

                        ssh -o StrictHostKeyChecking=no \
                            ${DEPLOY_USER}@${TOMCAT_HOST} \
                            "${START_SCRIPT}"

                        echo "Tomcat started."
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {

                    sshagent(credentials: [env.SSH_CREDENTIALS]) {

                        sh '''
                            echo "======================================"
                            echo "Verifying deployment"
                            echo "======================================"

                            echo "Waiting for application..."
                            sleep 15

                            curl -f \
                                http://${TOMCAT_HOST}:${TOMCAT_PORT}/${APP_NAME}/

                            echo ""
                            echo "======================================"
                            echo "Deployment successful!"
                            echo "======================================"
                        '''
                    }
                }
            }
        }
    }

    post {

        success {
            echo "Deployment completed successfully."
            echo "Instance: ${params.TOMCAT_INSTANCE}"
        }

        failure {
            echo "Deployment failed!"
            echo "Instance: ${params.TOMCAT_INSTANCE}"
        }

        always {
            cleanWs()
        }
    }
}
