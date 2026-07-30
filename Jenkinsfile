pipeline {

    agent any

    options {
        timestamps()
        ansiColor('xterm')
    }

    environment {

        FASTAPI = "http://127.0.0.1:8000"

        PYTHON = "python"

        PRISM_PORT = "4010"

        TARGET_HOST = ""

        OPENAPI_INPUT = ""

    }

    stages {

        stage('Checkout') {

            steps {

                checkout scm

            }

        }

        stage('Install Python Dependencies') {

            steps {

                bat """
                %PYTHON% -m pip install -r requirements.txt
                """

            }

        }

        stage('Install Prism') {

            steps {

                bat """
                npm install -g @stoplight/prism-cli
                """

            }

        }

        stage('Locate OpenAPI') {

            steps {

                script {

                    if (fileExists("openapi.yaml")) {

                        env.OPENAPI_INPUT = "openapi.yaml"

                    }

                    else if (fileExists("openapi.yml")) {

                        env.OPENAPI_INPUT = "openapi.yml"

                    }

                    else if (fileExists("swagger.yaml")) {

                        env.OPENAPI_INPUT = "swagger.yaml"

                    }

                    else if (fileExists("swagger.yml")) {

                        env.OPENAPI_INPUT = "swagger.yml"

                    }

                    else if (fileExists("swagger.json")) {

                        env.OPENAPI_INPUT = "swagger.json"

                    }

                    else if (fileExists("openapi.zip")) {

                        env.OPENAPI_INPUT = "openapi.zip"

                    }

                    else if (fileExists("api")) {

                        env.OPENAPI_INPUT = "api"

                    }

                    else {

                        error("Aucune spécification OpenAPI trouvée.")

                    }

                    echo "OpenAPI détecté : ${env.OPENAPI_INPUT}"

                }

            }

        }

        stage('Start Prism') {

            when {

                expression {

                    return env.TARGET_HOST.trim() == ""

                }

            }

            steps {

                script {

                    if (env.OPENAPI_INPUT.endsWith(".yaml") ||
                        env.OPENAPI_INPUT.endsWith(".yml") ||
                        env.OPENAPI_INPUT.endsWith(".json")) {

                        bat """
                        start "" prism mock ${env.OPENAPI_INPUT} --host 127.0.0.1 --port ${env.PRISM_PORT}
                        """

                        bat "timeout /t 8"

                    }

                    else {

                        echo "Prism ignoré (ZIP ou dossier)."

                    }

                }

            }

        }

        stage('Generate Scenario') {

            steps {

                script {

                    def curlCmd = """
                    curl ^
                    -X POST ^
                    """

                    if (fileExists(env.OPENAPI_INPUT)) {

                        curlCmd += """
                        -F "file=@${env.OPENAPI_INPUT}" ^
                        """

                    }
                    else {

                        curlCmd += """
                        -F "folder=@${env.OPENAPI_INPUT}" ^
                        """

                    }

                    if (env.TARGET_HOST.trim() != "") {

                        curlCmd += """
                        -F "target_host=${env.TARGET_HOST}" ^
                        """

                    }

                    curlCmd += """
                    ${env.FASTAPI}/generateScenario
                    """

                    bat curlCmd

                }

            }

        }

        stage('Execute JMeter') {

            steps {

                echo "Le backend FastAPI exécute automatiquement JMeter."

            }

        }

        stage('Publish Report') {

            steps {

                publishHTML(target: [

                    allowMissing: true,

                    alwaysLinkToLastBuild: true,

                    keepAll: true,

                    reportDir: 'generated/latest/report',

                    reportFiles: 'index.html',

                    reportName: 'Performance Report'

                ])

            }

        }

    }

    post {

        always {

            bat '''
            taskkill /F /IM prism.exe >nul 2>nul
            taskkill /F /IM node.exe >nul 2>nul
            '''

            archiveArtifacts artifacts: 'generated/**/*', fingerprint: true

        }

        success {

            echo "Pipeline terminé avec succès."

        }

        failure {

            echo "Le pipeline a échoué."

        }

    }

}
