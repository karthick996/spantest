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

                        // Record the output for this stage
                        currentBuild.description = formattedOutput

                        // Prompt for user confirmation to proceed
                        def userInput = input(
                            message: "Proceed to the next stage? Here is the Gitleaks output:\n\n${formattedOutput}",
                            parameters: [choice(name: 'Proceed', choices: ['Yes', 'No'], description: 'Choose whether to proceed to the next stage or not')]
                        )

                        if (userInput == 'No') {
                            error('Pipeline stopped by user')
                        }
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
                sh """${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=to-do-app \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=http://54.200.102.39:9000/ \
                    -Dsonar.login=squ_8b555ee4d028627d8434aa296467cca5c27b50ad"""
            }
        }
        
        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'DP'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Docker Build') {
            steps {
               script{
                   withDockerRegistry(credentialsId: 'docker-creds') {
                    sh "docker build -t todoapp:latest -f backend/Dockerfile . "
                    sh "docker tag todoapp:latest karthick996/todoapp:latest "
                 }
               }
            }
        }

        stage('Docker Push') {
            steps {
               script{
                   withDockerRegistry(credentialsId: 'docker-creds') {
                    sh "docker push karthick996/todoapp:latest "
                 }
               }
            }
        }

        stage('trivy') {
            steps {
               sh "trivy image karthick996/todoapp:latest"
            }
        }
    }
}
