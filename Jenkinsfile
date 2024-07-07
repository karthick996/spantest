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

                        // Proceed if no findings or if user confirms
                        if (parsedReport.isEmpty()) {
                            echo "No findings in the Gitleaks report."
                        } else {
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
                            if (userInput['PROCEED'] != 'Yes') {
                                currentBuild.result = 'ABORTED'
                                error('Pipeline aborted by user choice.')
                            }
                        }
                        currentBuild.description += "\n\nGitleaks Output:\n${gitleaksOutput}"
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
                    def sonarOutput = ''
                    try {
                        echo "Starting Sonar analysis..."
                        sonarOutput = sh(script: """${SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=to-do-app \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://18.237.125.100:9000 \
                            -Dsonar.login=squ_33e42a30bbce8e136861701ac4ce985839f4e460""", returnStdout: true).trim()
                        echo "Sonar analysis completed:\n${sonarOutput}"
                        currentBuild.description += "\n\nSonar Analysis Output:\n${sonarOutput}"
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
                    def dockerBuildOutput = ''
                    try {
                        echo "Building Docker image..."
                        withDockerRegistry(credentialsId: 'docker-creds') {
                            dockerBuildOutput = sh(script: "docker build -t todoapp:latest -f backend/Dockerfile .", returnStdout: true).trim()
                            sh "docker tag todoapp:latest karthick996/todoapp:latest"
                        }
                        echo "Docker build completed:\n${dockerBuildOutput}"
                        currentBuild.description += "\n\nDocker Build Output:\n${dockerBuildOutput}"
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
                    def dockerPushOutput = ''
                    try {
                        echo "Pushing Docker image to registry..."
                        withDockerRegistry(credentialsId: 'docker-creds') {
                            dockerPushOutput = sh(script: "docker push karthick996/todoapp:latest", returnStdout: true).trim()
                        }
                        echo "Docker push completed:\n${dockerPushOutput}"
                        currentBuild.description += "\n\nDocker Push Output:\n${dockerPushOutput}"
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
                    def trivyOutput = ''
                    try {
                        echo "Running Trivy scan..."
                        trivyOutput = sh(script: "trivy image karthick996/todoapp:latest", returnStdout: true).trim()
                        echo "Trivy scan completed:\n${trivyOutput}"
                        currentBuild.description += "\n\nTrivy Scan Output:\n${trivyOutput}"
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
                echo "Build summary:\n${currentBuild.description}"
            }
        }
    }
}
