pipeline {
    agent any

    environment {
        // ── SafeShip configuration ─────────────────────────────────────────
        SAFESHIP_EC2_IP     = '54.89.160.150'                     // Replace with your EC2 IP
        SAFESHIP_TENANT_ID  = '318997eaa6124b6d'
        SAFESHIP_API_KEY    = 'bcc6f5f5c2ce4b96971d9a2529620afa'
        SAFESHIP_THRESHOLD  = '70'                              // Block if score >= this
    }

    stages {

        // ─── 1. Checkout ────────────────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                echo "✅ Checked out: ${env.GIT_BRANCH} @ ${env.GIT_COMMIT?.take(7)}"
            }
        }

        // ─── 2. Compute diff metadata ────────────────────────────────────────
        stage('Compute Diff Size') {
            steps {
                script {
                    // Count changed lines vs. previous commit
                    def diffStat = sh(
                        script: "git diff --stat HEAD~1 HEAD 2>/dev/null | tail -1 || echo '0 files changed, 0 insertions(+), 0 deletions(-)'",
                        returnStdout: true
                    ).trim()

                    def added    = (diffStat =~ /(\d+) insertion/)  ? (diffStat =~ /(\d+) insertion/)[0][1].toInteger()  : 0
                    def deleted  = (diffStat =~ /(\d+) deletion/)   ? (diffStat =~ /(\d+) deletion/)[0][1].toInteger()   : 0
                    env.GIT_DIFF_SIZE = (added + deleted).toString()

                    echo "📊 Diff size: +${added} / -${deleted} = ${env.GIT_DIFF_SIZE} lines total"
                }
            }
        }

        // ─── 3. SafeShip Risk Check ──────────────────────────────────────────
        stage('SafeShip Risk Check') {
            steps {
                script {
                    def now     = new Date()
                    def hourVal = now.hours
                    def dayVal  = now.day   // 0 = Sunday … 6 = Saturday

                    echo "🚢 SafeShip: scoring deploy at hour=${hourVal}, day=${dayVal}, diff=${env.GIT_DIFF_SIZE} lines"

                    def payload = """{
                        "tenant_id":           "${env.SAFESHIP_TENANT_ID}",
                        "api_key":             "${env.SAFESHIP_API_KEY}",
                        "hour_of_day":         ${hourVal},
                        "day_of_week":         ${dayVal},
                        "diff_size":           ${env.GIT_DIFF_SIZE ?: 100},
                        "recent_failure_rate": 0.0
                    }"""

                    def res = sh(
                        script: """curl -s --max-time 10 -X POST http://${env.SAFESHIP_EC2_IP}/score \\
                            -H 'Content-Type: application/json' \\
                            -d '${payload}'""",
                        returnStdout: true
                    ).trim()

                    echo "SafeShip raw response: ${res}"

                    def result = readJSON text: res

                    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
                    echo "🎯 SafeShip Score : ${result.score} / 100"
                    echo "📋 Verdict        : ${result.verdict}"
                    // if (result.top_reasons) {
                    //     echo "⚠️  Top reasons    : ${result.top_reasons}"
                    // }
                    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

                    // Store score for downstream stages
                    // env.SAFESHIP_SCORE   = result.score.toString()
                    // env.SAFESHIP_VERDICT = result.verdict
                    env.SAFESHIP_SCORE   = result.score.toString()
env.SAFESHIP_VERDICT = result.verdict
env.DG_BUILD_ID      = result.build_id ?: ""

                    if (result.verdict == 'BLOCKED') {
                        error("🚫 SafeShip BLOCKED this deploy (score ${result.score}/100). Fix the risk factors and retry.")
                    } else if (result.verdict == 'CAUTION') {
                        currentBuild.description = "⚠️ SafeShip CAUTION: ${result.score}/100"
                        echo "⚠️  CAUTION mode — pipeline continues but review is recommended."
                    } else {
                        currentBuild.description = "✅ SafeShip SAFE: ${result.score}/100"
                        echo "✅ SafeShip says SAFE — proceeding with deploy."
                    }
                }
            }
        }

        // ─── 4. Build / Validate ────────────────────────────────────────────
        stage('Build') {
            steps {
                echo "🔨 Running build steps..."
                // Add your real build commands here, for example:
                //   sh 'npm ci && npm run build'
                //   sh 'mvn clean package -DskipTests'
                sh 'echo "Build complete ✅"'
            }
        }

        // ─── 5. Test ────────────────────────────────────────────────────────
        stage('Test') {
            steps {
                echo "🧪 Running test suite..."
                // Add your real test commands here, for example:
                //   sh 'npm test -- --watchAll=false'
                //   sh 'mvn test'
                sh 'echo "Tests passed ✅"'
            }
        }

        // ─── 6. Deploy (only runs if SafeShip didn't block) ─────────────────
        stage('Deploy') {
            when {
                expression { env.SAFESHIP_VERDICT != 'BLOCKED' }
            }
            steps {
                echo "🚀 Deploying... SafeShip score was ${env.SAFESHIP_SCORE}/100"
                // Add your real deploy commands here
                sh 'echo "Deployed successfully ✅"'
            }
        }
    }
post {
    success {
        echo "✅ Pipeline PASSED — SafeShip score: ${env.SAFESHIP_SCORE ?: 'N/A'}/100"
    }

    failure {
        echo "❌ Pipeline FAILED — SafeShip verdict: ${env.SAFESHIP_VERDICT ?: 'N/A'}"
    }

    always {
        script {
            echo "Pipeline finished. Branch: ${env.GIT_BRANCH} Commit: ${env.GIT_COMMIT?.take(7)}"

            if (env.DG_BUILD_ID?.trim()) {

                def finalLabel = (currentBuild.currentResult == 'SUCCESS') ? 0 : 1
                def sourceTxt  = (finalLabel == 0) ? "safe" : "failure"

                def logPayload = """{
                    "tenant_id":"${env.SAFESHIP_TENANT_ID}",
                    "api_key":"${env.SAFESHIP_API_KEY}",
                    "build_id":"${env.DG_BUILD_ID}",
                    "label":${finalLabel},
                    "label_source":"${sourceTxt}"
                }"""

                echo "📡 Sending outcome to /log for ${env.DG_BUILD_ID}"

                sh """
                curl -s -X POST http://${env.SAFESHIP_EC2_IP}/log \
                  -H 'Content-Type: application/json' \
                  -d '${logPayload}'
                """
            } else {
                echo "⚠️ No build_id found. Skipping /log"
            }
        }
    }
}