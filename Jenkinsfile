pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'SonarQube scanner'
        SLACK_CHANNEL = "#git-leaks-alerts"
        GITLEAKS_REPORT_FILE = 'gitleaks-report.json'
    }

    stages {
        stage('Run Gitleaks') {
            steps {
                script {
                    def proceed = false
                    try {
                        // Run Gitleaks and capture the output
                        def status = sh(script: "gitleaks detect --source . --report-format json --report-path ${GITLEAKS_REPORT_FILE} > gitleaks-output.txt 2>&1", returnStatus: true)

                        // Read the Gitleaks output file
                        def outputFileContent = readFile('gitleaks-output.txt')

                        // Read the JSON report file using readJSON step
                        def parsedReport = readJSON file: GITLEAKS_REPORT_FILE

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
                        ${outputFileContent}

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
                    } catch (Exception e) {
                        currentBuild.result = 'UNSTABLE'
                        echo 'Error running Gitleaks or displaying input.'
                    }

                    // Check the proceed variable before moving to the next stage
                    if (!proceed) {
                        currentBuild.result = 'ABORTED'
                        error('Pipeline aborted by user choice.')
                    }
                }
            }
        }
        stage('Git Checkout') {
            steps {
                git branch: 'main', changelog: false, poll: false, url: 'https://github.com/karthick996/spantest.git'
            }
        }

        

        stage('Sonar Analysis') {
            steps {
                sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=to-do-app \
                -Dsonar.sources=. \
                -Dsonar.host.url=http://35.91.88.89:9000 \
                -Dsonar.login=squ_6b0fe09c64d9409e1f0921c9d7ef591ef2a0ea21 '''
            }
        }

         stage('OWASP Dependency Check') {
            steps {
               dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'DP'
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
    }
}
