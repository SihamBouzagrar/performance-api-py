pipeline {

    agent any

    environment {

        AI_SERVER = "http://192.168.1.20:8000"      // Adresse du serveur IA
        OPENAPI_FILE = "openapi.yaml"

        POLL_INTERVAL = 10
        POLL_TIMEOUT = 1800

    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify OpenAPI') {
            steps {

                script {

                    if (!fileExists(env.OPENAPI_FILE)) {
                        error "openapi.yaml introuvable"
                    }

                    echo "Specification OpenAPI trouvée"

                }

            }
        }

        stage('Generate JMeter Scenario') {

            steps {

                script {

                    bat """
                    curl -X POST ^
                    -F file=@%OPENAPI_FILE% ^
                    %AI_SERVER%/generateScenario ^
                    -o response.json
                    """

                    def json = readJSON file: "response.json"

                    env.JOB_ID = json.jobId

                    echo "JOB = ${env.JOB_ID}"

                }

            }

        }

        stage('Wait Execution') {

            steps {

                script {

                    int elapsed = 0

                    while (elapsed < env.POLL_TIMEOUT.toInteger()) {

                        sleep env.POLL_INTERVAL.toInteger()

                        elapsed += env.POLL_INTERVAL.toInteger()

                        bat """
                        curl %AI_SERVER%/status/%JOB_ID% -o status.json
                        """

                        def status = readJSON file: "status.json"

                        echo "Status : ${status.status}"

                        if(status.status=="COMPLETED"){

                            echo "Execution terminée"

                            break

                        }

                        if(status.status=="FAILED"){

                            error status.error

                        }

                    }

                    if(elapsed>=env.POLL_TIMEOUT.toInteger()){

                        error "Timeout"

                    }

                }

            }

        }

        stage('Download Results') {

            steps {

                bat """
                curl %AI_SERVER%/download/%JOB_ID%/jtl -o result.jtl
                """

                bat """
                curl %AI_SERVER%/download/%JOB_ID%/report -o report.zip
                """

                bat """
                curl %AI_SERVER%/download/%JOB_ID%/log -o jmeter.log
                """

                powershell """
                if(Test-Path report.zip){
                    Expand-Archive report.zip report -Force
                }
                """

            }

        }

        stage('Archive') {

            steps {

                archiveArtifacts artifacts: '**/*.jtl', allowEmptyArchive: true

                archiveArtifacts artifacts: '**/*.log', allowEmptyArchive: true

                archiveArtifacts artifacts: 'report/**', allowEmptyArchive: true

            }

        }

    }

    post {

        always {

            echo "================================"

            echo "JOB ID : ${env.JOB_ID}"

            echo "Pipeline terminé"

            echo "================================"

        }

        success {

            echo "SUCCESS"

        }

        failure {

            echo "FAILED"

        }

    }

}
