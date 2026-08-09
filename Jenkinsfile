// =========================================================================
// Jenkinsfile — Repo "métier" (API à tester)
// Ce pipeline NE fait QUE :
//   1. Localiser la spec OpenAPI du repo
//   2. Appeler la plateforme AI Performance Testing (déjà déployée en continu,
//      repo séparé — voir README-architecture.md)
//   3. Attendre la fin du job, publier le rapport, archiver
//
// ⚠️ Threads / rampUp / loops / think-time / poids des transactions NE SONT
// PAS fixés ici : c'est l'Agent 4 (LLM Groq, via ScenarioGenerator) qui les
// détermine à partir de l'analyse de la spec (voir section 17.1.D du rapport).
// Imposer ces valeurs depuis Jenkins reviendrait à contourner l'IA.
// =========================================================================

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
        // Uniquement des choix d'INFRASTRUCTURE (où taper), jamais de scénario de test.
        string(
            name: 'TARGET_HOST',
            defaultValue: '',
            description: 'URL cible réelle (laisser vide = auto-résolution par TargetResolver / Prism mock si absente des servers OpenAPI)'
        )
        booleanParam(
            name: 'FORCE_MOCK',
            defaultValue: false,
            description: 'Forcer l\'utilisation du mock Prism même si une baseUrl réelle existe'
        )
    }

    environment {
        // URL fixe de la plateforme IA (autre conteneur Docker, port publié sur l'hôte).
        // host.docker.internal permet à Jenkins (conteneur) de joindre l'hôte Windows/Mac,
        // où le port 8000 du conteneur ai-performance-platform est publié.
        AI_SERVER_URL = 'http://host.docker.internal:8000'
        GENERATED_DIR = "${WORKSPACE}/generated"
        SPEC_FILE     = ""   // renseigné dynamiquement
        JOB_ID        = ""   // renseigné dynamiquement
    }

    triggers {
        githubPush()
    }

    stages {

        // ---------------------------------------------------------------
        stage('Checkout') {
            steps {
                echo "🔄 Récupération du dépôt..."
                checkout scm
                sh 'git rev-parse HEAD'
            }
        }

        // ---------------------------------------------------------------
        stage('Locate OpenAPI Spec') {
            steps {
                echo "🔍 Localisation de la spécification OpenAPI..."
                script {
                    def candidates = [
                        'openapi.yaml', 'openapi.yml', 'openapi.json',
                        'swagger.yaml', 'swagger.yml', 'swagger.json',
                        'api-spec.yaml', 'api-spec.json',
                        'petstore.json', 'petstore.yaml', 'petstore.yml',
                        'petstrore.json', 'petstrore.yaml', 'petstrore.yml'  // nom exact utilisé dans le repo
                    ]
                    def found = candidates.find { fileExists(it) }

                    if (!found) {
                        def result = sh(
                            script: "find . -maxdepth 3 -iregex '.*\\(openapi\\|swagger\\|petstore\\|petstrore\\|api-spec\\).*\\.\\(ya?ml\\|json\\)' | head -n 1",
                            returnStdout: true
                        ).trim()
                        found = result ?: null
                    }

                    if (!found) {
                        error("❌ Aucune spécification OpenAPI/Swagger trouvée dans le dépôt.")
                    }

                    env.SPEC_FILE = found
                    echo "✅ Spécification trouvée : ${env.SPEC_FILE}"
                }
            }
        }

        // ---------------------------------------------------------------
        stage('Check AI Platform Availability') {
            steps {
                echo "🔎 Vérification que la plateforme IA (service partagé) est disponible..."
                sh '''
                    set -e
                    if ! curl -sf "${AI_SERVER_URL}/health" > /dev/null; then
                        echo "❌ Plateforme AI Performance Testing injoignable sur ${AI_SERVER_URL}"
                        echo "   -> Vérifiez qu'elle est bien déployée en continu (voir repo dédié)."
                        exit 1
                    fi
                    echo "✅ Plateforme IA disponible."
                '''
            }
        }

        // ---------------------------------------------------------------
        stage('Generate & Run Scenario (LLM decides everything)') {
            steps {
                echo "🤖 Envoi de la spec — le LLM détermine seul threads/rampUp/loops/poids/assertions..."
                script {
                    def targetHostArg = params.TARGET_HOST?.trim() ?
                        "-F target_host=${params.TARGET_HOST.trim()}" : ""
                    def forceMockArg = params.FORCE_MOCK ? "-F use_mock=true" : ""

                    // Aucun -F threads=... / rampUp=... / loops=... :
                    // on laisse volontairement l'Agent 4 (Groq) décider.
                    def response = sh(
                        script: """
                            curl -s -X POST "${AI_SERVER_URL}/generateScenario" \
                                -F "file=@${env.SPEC_FILE}" \
                                ${targetHostArg} \
                                ${forceMockArg}
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Réponse brute : ${response}"

                    def jobId = sh(
                        script: "echo '${response}' | python3 -c \"import sys, json; print(json.load(sys.stdin).get('jobId', ''))\"",
                        returnStdout: true
                    ).trim()

                    if (!jobId) {
                        error("❌ Échec de la génération du scénario (jobId introuvable). Réponse: ${response}")
                    }

                    env.JOB_ID = jobId
                    echo "✅ Job créé : ${env.JOB_ID} — configuration décidée par l'IA, à consulter dans le rapport."
                }
            }
        }

        // ---------------------------------------------------------------
        stage('Wait for Completion') {
            steps {
                echo "⏳ Attente de la fin du job ${env.JOB_ID}..."
                script {
                    def status = 'QUEUED'
                    def maxAttempts = 300 // ~25 min à 5s d'intervalle
                    def attempt = 0

                    while (status in ['QUEUED', 'RUNNING'] && attempt < maxAttempts) {
                        sleep(time: 5, unit: 'SECONDS')
                        status = sh(
                            script: """
                                curl -s "${AI_SERVER_URL}/status/${env.JOB_ID}" \
                                    | python3 -c "import sys, json; print(json.load(sys.stdin).get('status', 'UNKNOWN'))"
                            """,
                            returnStdout: true
                        ).trim()
                        attempt++
                        echo "   Statut (${attempt}) : ${status}"
                    }

                    if (status != 'COMPLETED') {
                        error("❌ Le job ${env.JOB_ID} s'est terminé avec le statut : ${status}")
                    }
                    echo "✅ Test de charge terminé avec succès."
                }
            }
        }

        // ---------------------------------------------------------------
        stage('Download Results') {
            steps {
                echo "📥 Téléchargement des résultats (JTL, rapport, logs)..."
                sh '''
                    set -e
                    mkdir -p "${GENERATED_DIR}/${JOB_ID}"
                    cd "${GENERATED_DIR}/${JOB_ID}"

                    curl -s -o report.zip   "${AI_SERVER_URL}/download/${JOB_ID}/report"
                    curl -s -o results.jtl  "${AI_SERVER_URL}/download/${JOB_ID}/jtl"
                    curl -s -o jmeter.log   "${AI_SERVER_URL}/download/${JOB_ID}/log"
                    curl -s -o stdout.log   "${AI_SERVER_URL}/download/${JOB_ID}/stdout"

                    unzip -o report.zip -d report/
                '''
            }
        }

        // ---------------------------------------------------------------
        stage('Publish HTML Report') {
            steps {
                echo "📊 Publication du rapport HTML JMeter..."
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

    // -------------------------------------------------------------------
    post {
        always {
            echo "📦 Archivage des artefacts (incl. la configuration décidée par l'IA)..."
            archiveArtifacts artifacts: 'generated/**', allowEmptyArchive: true, fingerprint: true
        }
        success {
            echo "✅ Pipeline terminé avec succès — Job ${env.JOB_ID}"
            // slackSend(color: 'good', message: "✅ Test de charge réussi — Job ${env.JOB_ID} (build ${env.BUILD_URL})")
        }
        failure {
            echo "❌ Le pipeline a échoué."
            // slackSend(color: 'danger', message: "❌ Échec du test de charge (build ${env.BUILD_URL})")
        }
        unstable {
            echo "⚠️ Pipeline instable — vérifier les seuils de performance (taux d'erreur, latences)."
        }
    }
}