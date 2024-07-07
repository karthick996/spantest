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
                git branch: 'main', changelog: false, poll: false, url: 'https://github.com/karthick996/spantest.git'
            }
        }

        stage('Run Gitleaks') {
            steps {
                script {
                    def proceed = false
                    try {
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

                        // Read and debug the JSON report file
                        if (fileExists("${GITLEAKS_REPORT_FILE}")) {
                            def parsedReport = readJSON file: "${GITLEAKS_REPORT_FILE}"
                            echo "Parsed Gitleaks Report: ${parsedReport}"

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

                            // Debug: Print detailedFindings to verify its content
                            echo "Detailed Findings: ${detailedFindings}"

                            // Format the output to be user-friendly
                            def formattedOutput = """
                            Gitleaks Scan Report:

                            Detailed Findings:
                            ${detailedFindings}
                            """

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
                        } else {
                            echo "Gitleaks report file not found: ${GITLEAKS_REPORT_FILE}"
                            error("Gitleaks report file not found: ${GITLEAKS_REPORT_FILE}")
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
                sh """${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=to-do-app \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=http://18.237.125.100:9000 \
                    -Dsonar.login=squ_33e42a30bbce8e136861701ac4ce985839f4e460"""
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
