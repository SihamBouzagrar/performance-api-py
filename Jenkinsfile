pipeline {
    agent any

    // ============================================================
    // OPTIONS
    // ============================================================
    options {
        disableConcurrentBuilds()
        timeout(time: 45, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '15'))
        ansiColor('xterm')
        timestamps()
    }

    // ============================================================
    // PARAMETERS
    // ============================================================
    parameters {
        string(
            name: 'TARGET_HOST',
            defaultValue: '',
            description: "URL cible réelle. Laisser vide = résolution automatique / Prism mock."
        )
        booleanParam(
            name: 'FORCE_MOCK',
            defaultValue: false,
            description: "Forcer l'utilisation du mock Prism."
        )
    }

    // ============================================================
    // ENVIRONMENT
    // ============================================================
    environment {
        // Jenkins tourne dans Docker : host.docker.internal permet
        // d'atteindre le serveur FastAPI exposé sur la machine hôte.
        AI_SERVER_URL = 'http://host.docker.internal:8000'
        GENERATED_DIR = "${WORKSPACE}/generated"
    }

    // ============================================================
    // TRIGGERS
    // ============================================================
    triggers {
        githubPush()
    }

    // ============================================================
    // STAGES
    // ============================================================
    stages {

        // --------------------------------------------------------
        stage('Checkout') {
            steps {
                echo "=================================================="
                echo "JENKINSFILE VERSION : 2026-08-10-V4"
                echo "=================================================="
                echo "Récupération du dépôt..."
                checkout scm
                sh '''
                    set -e
                    echo "--- GIT REMOTE ---"
                    git remote -v
                    echo "--- GIT BRANCH ---"
                    git branch --show-current
                    echo "--- GIT COMMIT ---"
                    git rev-parse HEAD
                    echo "--- FILES ---"
                    ls -lah
                '''
            }
        }

        // --------------------------------------------------------
        stage('Locate OpenAPI Spec') {
            steps {
                script {
                    echo "Recherche de la spécification OpenAPI..."

                    def candidates = [
                        'openapi.yaml', 'openapi.yml', 'openapi.json',
                        'swagger.yaml', 'swagger.yml', 'swagger.json',
                        'api-spec.yaml', 'api-spec.yml', 'api-spec.json',
                        'petstore.json', 'petstore.yaml', 'petstore.yml',
                        'petstrore.json', 'petstrore.yaml', 'petstrore.yml'
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
                            script: '''
                                find . -maxdepth 4 -type f \\
                                    \\( -iname "*openapi*" -o -iname "*swagger*" -o -iname "*petstore*" -o -iname "*petstrore*" -o -iname "*api-spec*" \\) \\
                                    \\( -iname "*.yaml" -o -iname "*.yml" -o -iname "*.json" \\) \\
                                    | head -n 1
                            ''',
                            returnStdout: true
                        ).trim()
                        if (result) {
                            found = result
                        }
                    }

                    if (!found) {
                        error("Aucune spécification OpenAPI/Swagger trouvée dans le dépôt (openapi.*, swagger.*, api-spec.*, petstore.*, petstrore.*).")
                    }

                    env.SPEC_FILE = found
                    echo "Spec trouvée : ${env.SPEC_FILE}"

                    sh """
                        set -e
                        test -f "${env.SPEC_FILE}"
                        echo "Fichier OpenAPI valide."
                        ls -lh "${env.SPEC_FILE}"
                    """

                    if (!env.SPEC_FILE?.trim()) {
                        error("Incohérence : SPEC_FILE vide juste après assignation.")
                    }
                }
            }
        }

        // --------------------------------------------------------
        stage('Check AI Platform Availability') {
            steps {
                echo "Vérification de la plateforme IA..."
                sh '''
                    set -e
                    echo "AI_SERVER_URL=${AI_SERVER_URL}"
                    curl -fS "${AI_SERVER_URL}/health" -o /tmp/ai-health
                    echo "Réponse AI Server :"
                    cat /tmp/ai-health
                    echo ""
                    echo "Plateforme IA disponible."
                '''
            }
        }

        // --------------------------------------------------------
        // Note : Prism n'est PAS un service Docker autonome ici.
        // D'après docker-compose.yml, seul le conteneur "fastapi"
        // expose le port 4010, et Prism y est démarré/arrêté par
        // PrismService uniquement pendant le traitement de
        // /generateScenario. Le tester avant cet appel échouerait
        // systématiquement (rien n'écoute encore sur 4010).
        // --------------------------------------------------------

        stage('Generate & Run Scenario (LLM decides everything)') {
            steps {
                echo "Envoi de la spécification au moteur IA..."
                script {
                    def spec = env.SPEC_FILE
                    echo "SPEC_FILE reçu = [${spec}]"

                    if (!spec || spec == 'null' || spec.trim() == '') {
                        error("SPEC_FILE est vide ou null. Le stage Locate OpenAPI Spec n'a pas correctement transmis la variable.")
                    }
                    if (!fileExists(spec)) {
                        error("Le fichier OpenAPI n'existe pas : ${spec}")
                    }
                    echo "Fichier envoyé : ${spec}"

                    def targetHostArg = params.TARGET_HOST?.trim() ? "-F target_host=${params.TARGET_HOST.trim()}" : ""
                    def forceMockArg  = params.FORCE_MOCK ? "-F use_mock=true" : ""

                    // Écriture de la réponse dans un fichier plutôt que capture
                    // directe en variable Groovy : évite tout problème de
                    // quoting shell/JSON sur des réponses volumineuses.
                    sh """
                        set -e
                        curl -fS -X POST "${AI_SERVER_URL}/generateScenario" \\
                            -F "file=@${spec}" \\
                            ${targetHostArg} \\
                            ${forceMockArg} \\
                            -o "${WORKSPACE}/generate_response.json"
                    """

                    def response = readFile("${WORKSPACE}/generate_response.json")
                    echo "=================================================="
                    echo "RÉPONSE PLATEFORME IA"
                    echo "=================================================="
                    echo response

                    def generatedJobId = sh(
                        script: """
                            python3 -c "import json; print(json.load(open('${WORKSPACE}/generate_response.json')).get('jobId',''))"
                        """,
                        returnStdout: true
                    ).trim()

                    if (!generatedJobId) {
                        error("Aucun jobId retourné par la plateforme IA. Réponse : ${response}")
                    }

                    env.JOB_ID = generatedJobId
                    echo "JOB CRÉÉ : ${env.JOB_ID}"
                }
            }
        }

        // --------------------------------------------------------
        stage('Wait for Completion') {
            steps {
                script {
                    def currentJobId = env.JOB_ID
                    if (!currentJobId) {
                        error("JOB_ID est vide.")
                    }

                    echo "Attente du job : ${currentJobId}"

                    def status = 'QUEUED'
                    def maxAttempts = 300
                    def attempt = 0

                    while (status in ['QUEUED', 'RUNNING'] && attempt < maxAttempts) {
                        sleep(time: 5, unit: 'SECONDS')

                        sh """
                            set -e
                            curl -fS "${AI_SERVER_URL}/status/${currentJobId}" -o "${WORKSPACE}/status_response.json"
                        """

                        status = sh(
                            script: """
                                python3 -c "import json; print(json.load(open('${WORKSPACE}/status_response.json')).get('status','UNKNOWN'))"
                            """,
                            returnStdout: true
                        ).trim()

                        attempt++
                        echo "Statut (${attempt}/${maxAttempts}) : ${status}"
                    }

                    if (status in ['QUEUED', 'RUNNING']) {
                        error("Timeout d'attente du job ${currentJobId}. Dernier statut : ${status}, tentatives : ${attempt}.")
                    }
                    if (status != 'COMPLETED') {
                        error("Job ${currentJobId} terminé avec un statut inattendu : ${status}.")
                    }

                    echo "TEST DE PERFORMANCE TERMINÉ — JOB_ID=${currentJobId}, STATUS=${status}"
                }
            }
        }

        // --------------------------------------------------------
        stage('Download Results') {
            steps {
                sh '''
                    set -e
                    mkdir -p "${GENERATED_DIR}/${JOB_ID}"
                    cd "${GENERATED_DIR}/${JOB_ID}"

                    curl -fS -o report.zip  "${AI_SERVER_URL}/download/${JOB_ID}/report"
                    echo "report.zip téléchargé."

                    curl -fS -o results.jtl "${AI_SERVER_URL}/download/${JOB_ID}/jtl"
                    echo "results.jtl téléchargé."

                    curl -fS -o jmeter.log  "${AI_SERVER_URL}/download/${JOB_ID}/log"
                    echo "jmeter.log téléchargé."

                    curl -fS -o stdout.log  "${AI_SERVER_URL}/download/${JOB_ID}/stdout"
                    echo "stdout.log téléchargé."

                    mkdir -p report
                    unzip -o report.zip -d report/

                    echo "--- FICHIERS GÉNÉRÉS ---"
                    find . -maxdepth 3 -type f -print

                    echo "--- TAILLE DES FICHIERS ---"
                    du -ah . | sort -h
                '''
            }
        }

        // --------------------------------------------------------
        stage('Verify Report') {
            steps {
                sh '''
                    set -e
                    REPORT_DIR="${GENERATED_DIR}/${JOB_ID}/report"

                    if [ ! -d "${REPORT_DIR}" ]; then
                        echo "Répertoire report absent."
                        exit 1
                    fi
                    if [ ! -f "${REPORT_DIR}/index.html" ]; then
                        echo "index.html absent. Contenu du répertoire :"
                        find "${REPORT_DIR}" -maxdepth 3 -type f -print
                        exit 1
                    fi
                    echo "Rapport HTML trouvé : ${REPORT_DIR}/index.html"
                '''
            }
        }

        // --------------------------------------------------------
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

    // ============================================================
    // POST ACTIONS
    // ============================================================
    post {
        always {
            echo "Archivage des artefacts..."
            archiveArtifacts artifacts: 'generated/**', allowEmptyArchive: true, fingerprint: true
        }
        success {
            echo "PIPELINE TERMINÉ AVEC SUCCÈS — JOB_ID=${env.JOB_ID}"
        }
        failure {
            echo "PIPELINE ÉCHOUÉ — JOB_ID=${env.JOB_ID ?: 'N/A'}"
        }
        unstable {
            echo "PIPELINE INSTABLE — JOB_ID=${env.JOB_ID ?: 'N/A'}"
        }
    }
}