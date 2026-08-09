pipeline {
agent any

```
// ============================================================
// OPTIONS
// ============================================================

options {
    disableConcurrentBuilds()

    timeout(
        time: 45,
        unit: 'MINUTES'
    )

    buildDiscarder(
        logRotator(
            numToKeepStr: '15'
        )
    )

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
        description: 'URL cible réelle. Laisser vide = résolution automatique / Prism mock.'
    )

    booleanParam(
        name: 'FORCE_MOCK',
        defaultValue: false,
        description: 'Forcer l’utilisation du mock Prism.'
    )
}

// ============================================================
// ENVIRONMENT
// ============================================================

environment {

    /*
     * Serveur IA / FastAPI
     *
     * Jenkins étant exécuté dans Docker,
     * host.docker.internal permet d'atteindre
     * le serveur exposé sur la machine hôte.
     */
    AI_SERVER_URL = 'http://host.docker.internal:8000'

    /*
     * Prism Mock Server
     */
    PRISM_URL = 'http://host.docker.internal:4010'

    /*
     * Répertoire des résultats Jenkins
     */
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

    // ========================================================
    // 1. CHECKOUT
    // ========================================================

    stage('Checkout') {

        steps {

            echo "=================================================="
            echo "🚨 JENKINSFILE VERSION : 2026-08-09-V3"
            echo "=================================================="

            echo "🔄 Récupération du dépôt..."

            checkout scm

            sh '''
                set -e

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


    // ========================================================
    // 2. LOCATE OPENAPI
    // ========================================================

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
                    'api-spec.yml',
                    'api-spec.json',

                    'petstore.json',
                    'petstore.yaml',
                    'petstore.yml',

                    'petstrore.json',
                    'petstrore.yaml',
                    'petstrore.yml'
                ]

                def found = null

                // ------------------------------------------------
                // Recherche directe à la racine
                // ------------------------------------------------

                for (candidate in candidates) {

                    if (fileExists(candidate)) {

                        found = candidate

                        break
                    }
                }

                // ------------------------------------------------
                // Recherche récursive
                // ------------------------------------------------

                if (!found) {

                    def result = sh(
                        script: '''
                            find . -maxdepth 4 -type f \
                            \\( \
                                -iname '*openapi*' \
                                -o -iname '*swagger*' \
                                -o -iname '*petstore*' \
                                -o -iname '*petstrore*' \
                                -o -iname '*api-spec*' \
                            \\) \
                            \\( \
                                -iname '*.yaml' \
                                -o -iname '*.yml' \
                                -o -iname '*.json' \
                            \\) \
                            | head -n 1
                        ''',
                        returnStdout: true
                    ).trim()

                    if (result) {

                        found = result
                    }
                }

                // ------------------------------------------------
                // Vérification
                // ------------------------------------------------

                if (!found) {

                    error(
                        '''
```

❌ Aucune spécification OpenAPI/Swagger trouvée.

Fichiers recherchés :

* openapi.yaml
* openapi.yml
* openapi.json
* swagger.yaml
* swagger.yml
* swagger.json
* api-spec.yaml
* api-spec.json
* petstore.json
* petstore.yaml
* petstore.yml
  '''
  )
  }

  ```
                echo "✅ Spec trouvée : ${found}"

                /*
                 * Variable disponible dans les stages suivants.
                 */
                env.SPEC_FILE = found

                echo "🔎 env.SPEC_FILE = [${env.SPEC_FILE}]"

                sh """
                    set -e

                    echo "📄 SPEC_FILE = [${env.SPEC_FILE}]"

                    test -f "${env.SPEC_FILE}"

                    echo "✅ Fichier OpenAPI valide."

                    echo "======================================"
                    echo "TAILLE DU FICHIER"
                    echo "======================================"

                    ls -lh "${env.SPEC_FILE}"
                """
            }
        }
    }


    // ========================================================
    // 3. CHECK PRISM
    // ========================================================

    stage('Check Prism Mock') {

        steps {

            echo "🔎 Vérification du mock Prism..."

            sh '''
                set -e

                echo "PRISM_URL=${PRISM_URL}"

                echo "======================================"
                echo "TEST PRISM"
                echo "======================================"

                curl -fS \
                    "${PRISM_URL}/user/login?username=user1&password=password" \
                    -o /tmp/prism-response

                echo ""
                echo "📨 Réponse Prism :"

                cat /tmp/prism-response

                echo ""
                echo "✅ Prism est disponible."
            '''
        }
    }


    // ========================================================
    // 4. CHECK AI SERVER
    // ========================================================

    stage('Check AI Platform Availability') {

        steps {

            echo "🔎 Vérification de la plateforme IA..."

            sh '''
                set -e

                echo "AI_SERVER_URL=${AI_SERVER_URL}"

                echo "======================================"
                echo "TEST AI SERVER"
                echo "======================================"

                curl -fS \
                    "${AI_SERVER_URL}/health" \
                    -o /tmp/ai-health

                echo ""
                echo "📨 Réponse AI Server :"

                cat /tmp/ai-health

                echo ""
                echo "✅ Plateforme IA disponible."
            '''
        }
    }


    // ========================================================
    // 5. GENERATE SCENARIO
    // ========================================================

    stage('Generate & Run Scenario (LLM decides everything)') {

        steps {

            echo "🤖 Envoi de la spécification au moteur IA..."

            script {

                def spec = env.SPEC_FILE

                echo "🔎 SPEC_FILE reçu = [${spec}]"

                // ------------------------------------------------
                // Vérification SPEC_FILE
                // ------------------------------------------------

                if (!spec ||
                    spec == 'null' ||
                    spec.trim() == '') {

                    error(
                        '''
  ```

❌ SPEC_FILE est vide ou null.

Le stage Locate OpenAPI Spec n'a pas correctement
transmis la variable au stage actuel.
'''
)
}

```
                // ------------------------------------------------
                // Vérification fichier
                // ------------------------------------------------

                if (!fileExists(spec)) {

                    error(
                        "❌ Le fichier OpenAPI n'existe pas : ${spec}"
                    )
                }

                echo "📄 Fichier envoyé : ${spec}"

                // ------------------------------------------------
                // Construction de la commande curl
                // ------------------------------------------------

                def curlCommand = """
                    curl -fS -X POST "${AI_SERVER_URL}/generateScenario" \
                        -F "file=@${spec}"
                """

                // ------------------------------------------------
                // TARGET_HOST optionnel
                // ------------------------------------------------

                if (params.TARGET_HOST?.trim()) {

                    echo "🎯 TARGET_HOST fourni : ${params.TARGET_HOST}"

                    curlCommand += """
                        -F "target_host=${params.TARGET_HOST.trim()}"
                    """
                }

                // ------------------------------------------------
                // FORCE_MOCK
                // ------------------------------------------------

                if (params.FORCE_MOCK) {

                    echo "🧪 FORCE_MOCK=true"

                    curlCommand += """
                        -F "use_mock=true"
                    """
                }

                echo "🚀 Appel du moteur IA..."

                // ------------------------------------------------
                // Appel API
                // ------------------------------------------------

                def response = sh(
                    script: curlCommand,
                    returnStdout: true
                ).trim()

                echo "=================================================="
                echo "📨 RÉPONSE PLATEFORME IA"
                echo "=================================================="

                echo response

                // ------------------------------------------------
                // Extraction du jobId
                // ------------------------------------------------

                def generatedJobId = sh(
                    script: """
                        printf '%s' '${response.replace("'", "'\\\\''")}' |
                        python3 -c '
```

import sys
import json

data = json.load(sys.stdin)

print(data.get("jobId", ""))
'
""",
returnStdout: true
).trim()

```
                // ------------------------------------------------
                // Vérification jobId
                // ------------------------------------------------

                if (!generatedJobId) {

                    error(
                        """
```

❌ Aucun jobId retourné par la plateforme IA.

Réponse reçue :

${response}
"""
)
}

```
                env.JOB_ID = generatedJobId

                echo "=================================================="
                echo "✅ JOB CRÉÉ"
                echo "=================================================="

                echo "JOB_ID = ${env.JOB_ID}"
            }
        }
    }


    // ========================================================
    // 6. WAIT FOR COMPLETION
    // ========================================================

    stage('Wait for Completion') {

        steps {

            script {

                def currentJobId = env.JOB_ID

                if (!currentJobId) {

                    error(
                        "❌ JOB_ID est vide."
                    )
                }

                echo "⏳ Attente du job : ${currentJobId}"

                def status = 'QUEUED'

                /*
                 * 300 × 5 secondes = 25 minutes maximum
                 */
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
                            curl -fS \
                            "${AI_SERVER_URL}/status/${currentJobId}" |
                            python3 -c '
```

import sys
import json

data = json.load(sys.stdin)

print(data.get("status", "UNKNOWN"))
'
""",
returnStdout: true
).trim()

```
                    attempt++

                    echo "⏳ Statut (${attempt}/${maxAttempts}) : ${status}"
                }

                // ------------------------------------------------
                // Timeout du job
                // ------------------------------------------------

                if (status in ['QUEUED', 'RUNNING']) {

                    error(
                        """
```

❌ Timeout d'attente du job.

JOB_ID : ${currentJobId}
Dernier statut : ${status}
Nombre de tentatives : ${attempt}
"""
)
}

```
                // ------------------------------------------------
                // Job échoué
                // ------------------------------------------------

                if (status != 'COMPLETED') {

                    error(
                        """
```

❌ Job ${currentJobId} terminé avec un statut inattendu :

${status}
"""
)
}

```
                echo "=================================================="
                echo "✅ TEST DE PERFORMANCE TERMINÉ"
                echo "=================================================="

                echo "JOB_ID = ${currentJobId}"
                echo "STATUS = ${status}"
            }
        }
    }


    // ========================================================
    // 7. DOWNLOAD RESULTS
    // ========================================================

    stage('Download Results') {

        steps {

            sh '''
                set -e

                echo "📦 Préparation du répertoire..."

                mkdir -p "${GENERATED_DIR}/${JOB_ID}"

                cd "${GENERATED_DIR}/${JOB_ID}"

                echo "======================================"
                echo "📥 DOWNLOAD REPORT"
                echo "======================================"

                curl -fS \
                    -o report.zip \
                    "${AI_SERVER_URL}/download/${JOB_ID}/report"

                echo "✅ report.zip téléchargé."

                echo "======================================"
                echo "📥 DOWNLOAD JTL"
                echo "======================================"

                curl -fS \
                    -o results.jtl \
                    "${AI_SERVER_URL}/download/${JOB_ID}/jtl"

                echo "✅ results.jtl téléchargé."

                echo "======================================"
                echo "📥 DOWNLOAD JMETER LOG"
                echo "======================================"

                curl -fS \
                    -o jmeter.log \
                    "${AI_SERVER_URL}/download/${JOB_ID}/log"

                echo "✅ jmeter.log téléchargé."

                echo "======================================"
                echo "📥 DOWNLOAD STDOUT"
                echo "======================================"

                curl -fS \
                    -o stdout.log \
                    "${AI_SERVER_URL}/download/${JOB_ID}/stdout"

                echo "✅ stdout.log téléchargé."

                // ------------------------------------------------
                // Extraction report
                // ------------------------------------------------

                echo "======================================"
                echo "📂 EXTRACTION REPORT"
                echo "======================================"

                mkdir -p report

                unzip -o report.zip \
                    -d report/

                echo "======================================"
                echo "📁 FICHIERS GÉNÉRÉS"
                echo "======================================"

                find . -maxdepth 3 -type f -print

                echo "======================================"
                echo "📊 TAILLE DES FICHIERS"
                echo "======================================"

                du -ah . | sort -h
            '''
        }
    }


    // ========================================================
    // 8. VERIFY REPORT
    // ========================================================

    stage('Verify Report') {

        steps {

            sh '''
                set -e

                REPORT_DIR="${GENERATED_DIR}/${JOB_ID}/report"

                echo "🔎 Vérification du rapport..."

                if [ ! -d "${REPORT_DIR}" ]; then

                    echo "❌ Répertoire report absent."

                    exit 1
                fi

                if [ ! -f "${REPORT_DIR}/index.html" ]; then

                    echo "❌ index.html absent."

                    echo "Contenu du répertoire :"

                    find "${REPORT_DIR}" -maxdepth 3 -type f -print

                    exit 1
                fi

                echo "✅ Rapport HTML trouvé :"
                echo "${REPORT_DIR}/index.html"
            '''
        }
    }


    // ========================================================
    // 9. PUBLISH HTML REPORT
    // ========================================================

    stage('Publish HTML Report') {

        steps {

            publishHTML(
                target: [

                    allowMissing: false,

                    alwaysLinkToLastBuild: true,

                    keepAll: true,

                    reportDir:
                        "generated/${env.JOB_ID}/report",

                    reportFiles:
                        'index.html',

                    reportName:
                        'JMeter Performance Report'
                ]
            )
        }
    }
}

// ============================================================
// POST ACTIONS
// ============================================================

post {

    always {

        echo "=================================================="
        echo "📦 ARCHIVAGE DES ARTEFACTS"
        echo "=================================================="

        archiveArtifacts(
            artifacts: 'generated/**',
            allowEmptyArchive: true,
            fingerprint: true
        )
    }

    success {

        echo "=================================================="
        echo "✅ PIPELINE TERMINÉ AVEC SUCCÈS"
        echo "=================================================="

        echo "JOB_ID = ${env.JOB_ID}"
    }

    failure {

        echo "=================================================="
        echo "❌ PIPELINE ÉCHOUÉ"
        echo "=================================================="

        echo "JOB_ID = ${env.JOB_ID ?: 'N/A'}"
    }

    unstable {

        echo "=================================================="
        echo "⚠️ PIPELINE INSTABLE"
        echo "=================================================="

        echo "JOB_ID = ${env.JOB_ID ?: 'N/A'}"
    }
}


}
