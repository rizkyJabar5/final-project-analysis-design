pipeline {
    agent any

    environment {
        BACKEND_BASE_URL = 'http://172.235.251.236:8080'
        BACKEND_API_URL  = "${BACKEND_BASE_URL}/api/v1/analyze"
        // API_TOKEN = credentials('backend-api-token')
    }

    options {
        timeout(time: 15, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Fetching the latest design from the Git repository...'
                checkout scm
            }
        }

        stage('Prepare Tooling') {
            steps {
                script {
                    sh '''
                        set -e
                        if command -v jq >/dev/null 2>&1; then
                            echo "jq is available: $(jq --version)"
                            exit 0
                        fi

                        mkdir -p .bin
                        if [ ! -x .bin/jq ]; then
                            ARCH=$(uname -m)
                            case "$ARCH" in
                                x86_64|amd64)   JQ_ARCH=linux-amd64 ;;
                                aarch64|arm64)  JQ_ARCH=linux-arm64 ;;
                                *) echo "Unsupported arch for jq auto-install: $ARCH"; exit 1 ;;
                            esac
                            echo "Downloading jq ($JQ_ARCH) to .bin/jq..."
                            curl -sSL --fail -o .bin/jq \
                                "https://github.com/jqlang/jq/releases/download/jq-1.7.1/jq-${JQ_ARCH}"
                            chmod +x .bin/jq
                        fi
                        echo "Local jq installed: $(.bin/jq --version)"
                    '''
                    // Prepend the workspace .bin dir so every later sh step sees jq.
                    env.PATH = "${env.WORKSPACE}/.bin:${env.PATH}"
                }
            }
        }

        stage('Validate Commit Files') {
            steps {
                script {
                    echo 'Looking for recently committed design files...'

                    // Only get files from the last commit
                    def files = sh(
                        script: "git diff-tree --no-commit-id --name-only -r HEAD | grep -E '\\.(xml|bpmn|puml|uml|jpg|jpeg|png|svg)\$' || true",
                        returnStdout: true
                    ).trim()

                    env.CHANGED_FILES = files

                    if (env.CHANGED_FILES == "") {
                        echo "No design files found in last commit. Skipping analysis."
                    } else {
                        echo "Design files to analyze:\n${env.CHANGED_FILES}"
                    }
                }
            }
        }

        stage('Analyze Designs') {
            when {
                expression { env.CHANGED_FILES != "" }
            }
            steps {
                script {
                    def fileArray = env.CHANGED_FILES.split('\n')
                    def pipelineFailed = false
                    def results = []   // Per-file rows for the final summary.

                    sh "mkdir -p reports"

                    for (int i = 0; i < fileArray.size(); i++) {
                        def file = fileArray[i].trim()
                        if (file == "") continue

                        echo "------------------------------------------------"
                        echo "Sending file to backend: ${file}"

                        // Body goes to response.json; HTTP code comes back via stdout.
                        def httpStatus = sh(
                            script: """
                                curl -s -o response.json -w '%{http_code}' \
                                     -X POST '${BACKEND_API_URL}' \
                                     -F 'design_file=@${file}'
                            """,
                            returnStdout: true
                        ).trim()

                        if (httpStatus != "200") {
                            echo "ERROR: Backend returned HTTP ${httpStatus}"
                            sh "cat response.json || true"
                            pipelineFailed = true
                            results << [file: file, status: "HTTP ${httpStatus}", score: "—", threats: "—", link: ""]
                            continue
                        }

                        // Defensive jq: tolerate missing fields so partial responses don't crash the build.
                        def isPassed      = sh(script: "jq -r '.passedQualityGate // false' response.json", returnStdout: true).trim()
                        def reportPdfPath = sh(script: "jq -r '.downloadLinks.pdf // empty'  response.json", returnStdout: true).trim()
                        def securityScore = sh(script: "jq -r '.score // 0'                  response.json", returnStdout: true).trim()
                        def totalThreats  = sh(script: "jq    '.threats | length'            response.json", returnStdout: true).trim()

                        def passed = (isPassed == "true")
                        def statusText = passed ? "PASSED" : "FAILED"

                        echo "Result: ${statusText} | Score: ${securityScore}/100 | Threats: ${totalThreats}"

                        // Quality gate failed → download the PDF report and stash it as a Jenkins artifact.
                        def pdfLink = ""
                        if (!passed && reportPdfPath) {
                            def pdfUrl   = "${BACKEND_BASE_URL}${reportPdfPath}"
                            def safeName = file.replaceAll('[^A-Za-z0-9._-]', '_')
                            def pdfFile  = "reports/${safeName}.pdf"

                            echo "Quality gate failed — downloading PDF report from ${pdfUrl}"
                            def pdfStatus = sh(
                                script: "curl -s -o '${pdfFile}' -w '%{http_code}' --max-time 300 '${pdfUrl}'",
                                returnStdout: true
                            ).trim()

                            if (pdfStatus == "200") {
                                pdfLink = "${env.BUILD_URL}artifact/${pdfFile}"
                                echo "PDF report archived: ${pdfLink}"
                            } else {
                                echo "WARNING: PDF download failed (HTTP ${pdfStatus}) — falling back to backend URL"
                                pdfLink = pdfUrl
                                sh "rm -f '${pdfFile}'"
                            }
                        }

                        results << [
                            file: file,
                            status: statusText,
                            score: "${securityScore}/100",
                            threats: totalThreats,
                            link: pdfLink
                        ]

                        if (!passed) {
                            pipelineFailed = true
                        }
                    }

                    // Final clickable-link summary printed to the build console.
                    echo "================================================"
                    echo "  STRIDE Analysis Summary"
                    echo "================================================"
                    for (r in results) {
                        echo ""
                        echo "• ${r.file}"
                        echo "    Status:  ${r.status}"
                        echo "    Score:   ${r.score}"
                        echo "    Threats: ${r.threats}"
                        if (r.link) {
                            echo "    Report:  ${r.link}"
                        }
                    }
                    echo ""
                    echo "================================================"

                    // Add the same summary to the build description so it
                    // shows clickable on the build page (Jenkins linkifies URLs).
                    def desc = "STRIDE Analysis Results:"
                    for (r in results) {
                        desc += "\n• ${r.file} — ${r.status} (score ${r.score}, ${r.threats} threats)"
                        if (r.link) {
                            desc += "\n  Report: ${r.link}"
                        }
                    }
                    currentBuild.description = desc

                    if (pipelineFailed) {
                        error("Pipeline stopped: Quality Gate FAILED — vulnerabilities found in one or more design files.")
                    } else {
                        echo "All designs passed the Quality Gate."
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/*.pdf', allowEmptyArchive: true, fingerprint: true
            sh "rm -f response.json"
        }
    }
}
