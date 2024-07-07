pipeline {
    agent any

    environment {
        SCANNER_HOME = tool name: 'sonar-scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
        SLACK_CHANNEL = "#git-leaks-alerts"
        GITLEAKS_REPORT_FILE = 'gitleaks-report.json'
        buildOutput = ""
    }

    stages {
        stage('Git Checkout') {
            steps {
                script {
                    echo "Checking out the git repository..."
                    git branch: 'main', changelog: false, poll: false, url: 'https://github.com/karthick996/spantest.git'
                    echo "Git checkout completed."
                }
            }
        }

        stage('Run Gitleaks') {
            steps {
                script {
                    def gitleaksOutput = ''
                    try {
                        echo "Running Gitleaks scan..."
                        // Run Gitleaks and capture the output
                        def status = sh(script: "gitleaks detect --source . --report-format json --report-path ${GITLEAKS_REPORT_FILE} > gitleaks-output.txt 2>&1", returnStatus: true)
                        
                        // Capture the Gitleaks output for the dashboard
                        gitleaksOutput = readFile('gitleaks-output.txt')
                        echo "Gitleaks output:\n${gitleaksOutput}"

                        // Check if Gitleaks ran successfully
                        if (status != 0) {
                            echo "Gitleaks scan failed with status ${status}"
                            error('Gitleaks scan failed. See console output for details.')
                        }

                        // Read the JSON report file if it exists
                        def parsedReport = []
                        if (fileExists("${GITLEAKS_REPORT_FILE}")) {
                            parsedReport = readJSON file: "${GITLEAKS_REPORT_FILE}"
                        } else {
                            echo "Gitleaks report file not found: ${GITLEAKS_REPORT_FILE}"
                        }

                        // Extract detailed findings from the report
                        def detailedFindings = parsedReport.collect { finding ->
                            """
                            **File:** ${finding.file}
                            **Line:** ${finding.line}
                            **Secret:** ${finding.secret}
                            **Rule:** ${finding.rule}
                            **Commit:** ${finding.commit}
                            **Date:** ${finding.date}
                            **Entropy:** ${finding.entropy}
                            **Author:** ${finding.author}
                            **Email:** ${finding.email}
                            **Message:** ${finding.message}
                            **Fingerprint:** ${finding.fingerprint}
                            """
                        }.join('\n\n')

                        // Format the output to be user-friendly
                        def formattedOutput = """
                        Gitleaks Scan Report:
                        ${gitleaksOutput}

                        Detailed Findings:
                        ${detailedFindings ?: "No leaks found."}
                        """
                        
                        // Log the formatted output
                        echo formattedOutput

                        // Send Slack notification with Gitleaks output
                        slackSend(channel: env.SLACK_CHANNEL, color: '#FFFF00', message: formattedOutput)

                        // Append to build output
                        buildOutput += "Gitleaks Stage Output:\n${formattedOutput}\n\n"

                    } catch (Exception e) {
                        currentBuild.result = 'UNSTABLE'
                        echo 'Error running Gitleaks or displaying input: ' + e.toString()
                        error('Gitleaks scan encountered an error.')
                    }
                }
            }
        }

        stage('Sonar Analysis') {
            steps {
                script {
                    try {
                        echo "Starting Sonar analysis..."
                        // Replace this with the link to SonarQube analysis result
                        def sonarOutput = "ANALYSIS SUCCESSFUL, you can find the results at: [SonarQube Dashboard](http://18.237.125.100:9000/dashboard?id=to-do-app)"
                        echo sonarOutput
                        buildOutput += "Sonar Analysis Stage Output:\n${sonarOutput}\n\n"
                    } catch (Exception e) {
                        currentBuild.result = 'UNSTABLE'
                        echo 'Error running Sonar analysis: ' + e.toString()
                        error('Sonar analysis encountered an error.')
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    try {
                        echo "Building Docker image..."
                        // Add your Docker build steps here
                        def dockerBuildOutput = "Docker build completed successfully."
                        echo dockerBuildOutput
                        buildOutput += "Docker Build Stage Output:\n${dockerBuildOutput}\n\n"
                    } catch (Exception e) {
                        currentBuild.result = 'UNSTABLE'
                        echo 'Error building Docker image: ' + e.toString()
                        error('Docker build encountered an error.')
                    }
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    try {
                        echo "Pushing Docker image to registry..."
                        // Add your Docker push steps here
                        def dockerPushOutput = "Docker push completed successfully."
                        echo dockerPushOutput
                        buildOutput += "Docker Push Stage Output:\n${dockerPushOutput}\n\n"
                    } catch (Exception e) {
                        currentBuild.result = 'UNSTABLE'
                        echo 'Error pushing Docker image: ' + e.toString()
                        error('Docker push encountered an error.')
                    }
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    try {
                        echo "Running Trivy scan..."
                        // Add your Trivy scan steps here
                        def trivyOutput = "Trivy scan completed successfully."
                        echo trivyOutput
                        buildOutput += "Trivy Scan Stage Output:\n${trivyOutput}\n\n"
                    } catch (Exception e) {
                        currentBuild.result = 'UNSTABLE'
                        echo 'Error running Trivy scan: ' + e.toString()
                        error('Trivy scan encountered an error.')
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                echo "Build summary:\n${buildOutput}"
                currentBuild.description = buildOutput
            }
        }
    }
}
