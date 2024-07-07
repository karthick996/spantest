pipeline {
    agent any

    environment {
        SCANNER_HOME = tool name: 'sonar-scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
        SLACK_CHANNEL = "#git-leaks-alerts"
        GITLEAKS_REPORT_FILE = 'gitleaks-report.json'
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
                    def proceed = false
                    try {
                        echo "Running Gitleaks scan..."
                        // Run Gitleaks and capture the output
                        def status = sh(script: "gitleaks detect --source . --report-format json --report-path ${GITLEAKS_REPORT_FILE} > gitleaks-output.txt 2>&1", returnStatus: true)
                        
                        // Log the Gitleaks output for debugging
                        def gitleaksOutput = readFile('gitleaks-output.txt')
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

                        // Check if the parsed report contains data
                        if (parsedReport.isEmpty()) {
                            echo "No findings in the Gitleaks report."
                            error('No findings in the Gitleaks report.')
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

                        // Prompt user for confirmation to proceed with Gitleaks findings
                        def userInput = input(
                            id: 'proceedToNextStage',
                            message: 'Gitleaks scan completed. Review the findings and decide whether to proceed.',
                            parameters: [
                                [$class: 'TextParameterDefinition', defaultValue: formattedOutput, description: 'Gitleaks findings', name: 'GITLEAKS_OUTPUT'],
                                [$class: 'ChoiceParameterDefinition', choices: ['Yes', 'No'], description: 'Proceed to next stage?', name: 'PROCEED']
                            ]
                        )

                        // Set proceed variable based on user input
                        if (userInput['PROCEED'] == 'Yes') {
                            proceed = true
                        }
                    } catch (Exception e) {
                        currentBuild.result = 'UNSTABLE'
                        echo 'Error running Gitleaks or displaying input: ' + e.toString()
                    }

                    // Check the proceed variable before moving to the next stage
                    if (!proceed) {
                        currentBuild.result = 'ABORTED'
                        error('Pipeline aborted by user choice.')
                    }
                }
            }
        }

        stage('Sonar Analysis') {
            steps {
                script {
                    echo "Starting Sonar analysis..."
                    sh """${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=to-do-app \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://18.237.125.100:9000 \
                        -Dsonar.login=squ_33e42a30bbce8e136861701ac4ce985839f4e460"""
                    echo "Sonar analysis completed."
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    echo "Building Docker image..."
                    withDockerRegistry(credentialsId: 'docker-creds') {
                        sh "docker build -t todoapp:latest -f backend/Dockerfile ."
                        sh "docker tag todoapp:latest karthick996/todoapp:latest"
                    }
                    echo "Docker build and tag completed."
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    echo "Pushing Docker image to registry..."
                    withDockerRegistry(credentialsId: 'docker-creds') {
                        sh "docker push karthick996/todoapp:latest"
                    }
                    echo "Docker push completed."
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    echo "Running Trivy scan..."
                    sh "trivy image karthick996/todoapp:latest"
                    echo "Trivy scan completed."
                }
            }
        }
    }
}
