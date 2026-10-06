pipeline {
    agent any

    environment {
        SONAR_HOST_URL   = 'http://sonarqube:9000'
        SONAR_TOKEN      = credentials('sonarqube-token')
        SLACK_CHANNEL    = '#pipeline-psw'
        SLACK_CREDENTIAL = 'slack-token'
        APP_NAME         = 'PSW-Pipeline-Base'
        APP_PORT         = '8085'
    }

    tools {
        jdk 'JDK-17'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                dir('PSW_Pipeline_Base') {
                    sh 'mvn clean verify -B'
                }
            }
            post {
                always {
                    junit testResults: 'PSW_Pipeline_Base/target/surefire-reports/*.xml',
                          allowEmptyResults: true
                    jacoco(
                        execPattern:      'PSW_Pipeline_Base/target/jacoco.exec',
                        classPattern:     'PSW_Pipeline_Base/target/classes',
                        sourcePattern:    'PSW_Pipeline_Base/src/main/java',
                        exclusionPattern: '**/*Application*,**/model/**'
                    )
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                dir('PSW_Pipeline_Base') {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            mvn sonar:sonar \
                              -Dsonar.projectKey=psw-pipeline-base \
                              -Dsonar.projectName='PSW Pipeline Base' \
                              -Dsonar.projectVersion=0.0.1 \
                              -Dsonar.host.url=${SONAR_HOST_URL} \
                              -Dsonar.token=${SONAR_TOKEN} \
                              -Dsonar.java.source=17 \
                              -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                              -Dsonar.exclusions=**/model/**,**/*Application* \
                              -B
                        """
                    }
                }
            }
            post {
                always {
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: false
                    }
                }
            }
        }

        stage('JMeter Load Test') {
            steps {
                sh 'mkdir -p PSW_Pipeline_Base/jmeter/results'
                dir('PSW_Pipeline_Base') {
                    sh 'nohup java -jar target/psw-pipeline-base-0.0.1-SNAPSHOT.jar --server.port=${APP_PORT} > /tmp/app.log 2>&1 &'
                    sh 'sleep 15'
                    sh """
                        jmeter -n \
                          -t jmeter/psw-load-test.jmx \
                          -l jmeter/results/results.jtl \
                          -e \
                          -o jmeter/results/html-report \
                          -Jhost=localhost \
                          -Jport=${APP_PORT} \
                          -Jthreads=75 \
                          -Jrampup=30 \
                          -Jduration=60
                    """
                }
            }
            post {
                always {
                    sh "pkill -f 'psw-pipeline-base' || true"
                    publishHTML(target: [
                        allowMissing:          false,
                        alwaysLinkToLastBuild: true,
                        keepAll:               true,
                        reportDir:             'PSW_Pipeline_Base/jmeter/results/html-report',
                        reportFiles:           'index.html',
                        reportName:            'JMeter Report'
                    ])
                    archiveArtifacts artifacts: 'PSW_Pipeline_Base/jmeter/results/results.jtl',
                                     allowEmptyArchive: true
                }
            }
        }

        stage('Notify Success') {
            steps {
                slackSend(
                    channel:           env.SLACK_CHANNEL,
                    color:             'good',
                    tokenCredentialId: env.SLACK_CREDENTIAL,
                    message:           """:white_check_mark: *Pipeline Exitoso* — `${APP_NAME}`
> *Branch:* `${env.GIT_BRANCH}`
> *Commit:* `${env.GIT_COMMIT?.take(7)}`
> *Build:* <${env.BUILD_URL}|#${env.BUILD_NUMBER}>
> *Duración:* ${currentBuild.durationString}"""
                )
            }
        }
    }

    post {
        failure {
            slackSend(
                channel:           env.SLACK_CHANNEL,
                color:             'danger',
                tokenCredentialId: env.SLACK_CREDENTIAL,
                message:           """:x: *Pipeline FALLIDO* — `${APP_NAME}`
> *Branch:* `${env.GIT_BRANCH}`
> *Commit:* `${env.GIT_COMMIT?.take(7)}`
> *Build:* <${env.BUILD_URL}|#${env.BUILD_NUMBER}>
> *Stage fallida:* `${env.STAGE_NAME}`
> Logs: ${env.BUILD_URL}console"""
            )
        }
        always {
            cleanWs()
        }
    }
}
