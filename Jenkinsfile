
pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timeout(time: 45, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '15'))
        ansiColor('xterm')
        timestamps()
    }

    parameters {
        string(
            name: 'TARGET_HOST',
            defaultValue: '',
            description: 'URL cible réelle (laisser vide = auto-résolution par TargetResolver / Prism mock)'
        )

        booleanParam(
            name: 'FORCE_MOCK',
            defaultValue: false,
            description: 'Forcer l’utilisation du mock Prism'
        )
    }

    environment {
        AI_SERVER_URL = 'http://host.docker.internal:8000'
        GENERATED_DIR = "${WORKSPACE}/generated"
    }

    triggers {
        githubPush()
    }

    stages {

        // ============================================================
       stage('Checkout') {
    steps {
        echo "🚨 JENKINSFILE VERSION TEST : 2026-08-09-V2"

        echo "🔄 Récupération du dépôt..."

        checkout scm

        sh '''
            echo "======================================"
            echo "GIT REMOTE"
            echo "======================================"
            git remote -v

            echo "======================================"
            echo "GIT BRANCH"
            echo "======================================"
            git branch --show-current

            echo "======================================"
            echo "GIT COMMIT"
            echo "======================================"
            git rev-parse HEAD

            echo "======================================"
            echo "FILES"
            echo "======================================"
            ls -lah
        '''
    }
}
        // ============================================================
        stage('Locate OpenAPI Spec') {
            steps {
                script {

                    echo "🔍 Recherche de la spécification OpenAPI..."

                    def candidates = [
                        'openapi.yaml',
                        'openapi.yml',
                        'openapi.json',
                        'swagger.yaml',
                        'swagger.yml',
                        'swagger.json',
                        'api-spec.yaml',
                        'api-spec.json',
                        'petstore.json',
                        'petstore.yaml',
                        'petstore.yml',
                        'petstrore.json',
                        'petstrore.yaml',
                        'petstrore.yml'
                    ]

                    def found = null

                    for (candidate in candidates) {
                        if (fileExists(candidate)) {
                            found = candidate
                            break
                        }
                    }

                    if (!found) {
                        def result = sh(
                            script: """
                                find . -maxdepth 3 -type f \
                                \\( -iname '*openapi*' \
                                -o -iname '*swagger*' \
                                -o -iname '*petstore*' \
                                -o -iname '*petstrore*' \
                                -o -iname '*api-spec*' \\) \
                                \\( -iname '*.yaml' \
                                -o -iname '*.yml' \
                                -o -iname '*.json' \\) \
                                | head -n 1
                            """,
                            returnStdout: true
                        ).trim()

                        if (result) {
                            found = result
                        }
                    }

                    if (!found) {
                        error("❌ Aucune spécification OpenAPI/Swagger trouvée.")
                    }

                    echo "✅ Spec trouvée : ${found}"

                    /*
                     * IMPORTANT :
                     * On stocke la valeur dans une variable d'environnement
                     * Jenkins pour les stages suivants.
                     */
                    env.SPEC_FILE = found

                    echo "🔎 Vérification env.SPEC_FILE = [${env.SPEC_FILE}]"

                    sh """
                        echo "📄 SPEC_FILE depuis shell = [${env.SPEC_FILE}]"
                        test -f "${env.SPEC_FILE}"
                    """
                }
            }
        }

        // ============================================================
        stage('Check AI Platform Availability') {
            steps {

                echo "🔎 Vérification de la plateforme IA..."

                sh '''
                    set -e

                    echo "AI_SERVER_URL=${AI_SERVER_URL}"

                    curl -sf "${AI_SERVER_URL}/health" > /dev/null

                    echo "✅ Plateforme IA disponible."
                '''
            }
        }

        // ============================================================
        stage('Generate & Run Scenario (LLM decides everything)') {
            steps {

                echo "🤖 Envoi de la spec au moteur IA..."

                script {

                    /*
                     * IMPORTANT :
                     * On récupère la valeur depuis env au début du stage.
                     */
                    def spec = env.SPEC_FILE

                    echo "🔎 SPEC_FILE reçu = [${spec}]"

                    if (!spec ||
                        spec == 'null' ||
                        spec.trim() == '') {

                        error("""
                        ❌ SPEC_FILE est vide/null.

                        Le stage Locate OpenAPI Spec n'a pas correctement
                        transmis la variable au stage actuel.
                        """)
                    }

                    if (!fileExists(spec)) {
                        error("❌ Le fichier OpenAPI n'existe pas : ${spec}")
                    }

                    echo "📄 Fichier envoyé : ${spec}"

                    def curlCommand = """
                        curl -sS -X POST "${AI_SERVER_URL}/generateScenario" \
                            -F "file=@${spec}"
                    """

                    if (params.TARGET_HOST?.trim()) {
                        curlCommand += """
                            -F "target_host=${params.TARGET_HOST.trim()}"
                        """
                    }

                    if (params.FORCE_MOCK) {
                        curlCommand += """
                            -F "use_mock=true"
                        """
                    }

                    def response = sh(
                        script: curlCommand,
                        returnStdout: true
                    ).trim()

                    echo "📨 Réponse plateforme IA : ${response}"

                    def generatedJobId = sh(
                        script: """
                            printf '%s' '${response.replace("'", "'\\\\''")}' |
                            python3 -c '
import sys
import json

data = json.load(sys.stdin)

print(data.get("jobId", ""))
'
                        """,
                        returnStdout: true
                    ).trim()

                    if (!generatedJobId) {
                        error("""
                        ❌ Aucun jobId retourné par la plateforme IA.

                        Réponse :
                        ${response}
                        """)
                    }

                    env.JOB_ID = generatedJobId

                    echo "✅ Job créé : ${env.JOB_ID}"
                }
            }
        }

        // ============================================================
        stage('Wait for Completion') {
            steps {

                script {

                    def currentJobId = env.JOB_ID

                    if (!currentJobId) {
                        error("❌ JOB_ID est vide.")
                    }

                    echo "⏳ Attente du job : ${currentJobId}"

                    def status = 'QUEUED'
                    def maxAttempts = 300
                    def attempt = 0

                    while (
                        status in ['QUEUED', 'RUNNING'] &&
                        attempt < maxAttempts
                    ) {

                        sleep(
                            time: 5,
                            unit: 'SECONDS'
                        )

                        status = sh(
                            script: """
                                curl -sS \
                                "${AI_SERVER_URL}/status/${currentJobId}" |
                                python3 -c '
import sys
import json

data = json.load(sys.stdin)
print(data.get("status", "UNKNOWN"))
'
                            """,
                            returnStdout: true
                        ).trim()

                        attempt++

                        echo "   Statut (${attempt}) : ${status}"
                    }

                    if (status != 'COMPLETED') {
                        error(
                            "❌ Job ${currentJobId} terminé avec statut : ${status}"
                        )
                    }

                    echo "✅ Test terminé."
                }
            }
        }

        // ============================================================
        stage('Download Results') {
            steps {

                sh '''
                    set -e

                    mkdir -p "${GENERATED_DIR}/${JOB_ID}"

                    cd "${GENERATED_DIR}/${JOB_ID}"

                    echo "📥 Téléchargement du rapport..."

                    curl -fS -o report.zip \
                        "${AI_SERVER_URL}/download/${JOB_ID}/report"

                    curl -fS -o results.jtl \
                        "${AI_SERVER_URL}/download/${JOB_ID}/jtl"

                    curl -fS -o jmeter.log \
                        "${AI_SERVER_URL}/download/${JOB_ID}/log"

                    curl -fS -o stdout.log \
                        "${AI_SERVER_URL}/download/${JOB_ID}/stdout"

                    mkdir -p report

                    unzip -o report.zip -d report/

                    echo "✅ Résultats téléchargés."
                '''
            }
        }

        // ============================================================
        stage('Publish HTML Report') {
            steps {

                publishHTML(target: [
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: "generated/${env.JOB_ID}/report",
                    reportFiles: 'index.html',
                    reportName: 'JMeter Performance Report'
                ])
            }
        }
    }

    // ================================================================
    post {

        always {

            echo "📦 Archivage des artefacts..."

            archiveArtifacts(
                artifacts: 'generated/**',
                allowEmptyArchive: true,
                fingerprint: true
            )
        }

        success {
            echo "✅ Pipeline terminé avec succès — Job ${env.JOB_ID}"
        }

        failure {
            echo "❌ Le pipeline a échoué."
        }

        unstable {
            echo "⚠️ Pipeline instable."
        }
    }
}
