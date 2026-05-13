pipeline {
    agent any

    environment {
        JMETER_HOME = "C:\\apache-jmeter-5.6.3\\apache-jmeter-5.6.3"
        JMETER = "${JMETER_HOME}\\bin\\jmeter.bat"
        JMX_FILE = "jpetstore_jenkins_comparision\\SCR01_Jpetstore.jmx"
        REPORT_NAME = "SCR01_Report_Build_${BUILD_NUMBER}"
        HISTORY_DIR = "C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\Jenkins_Comparision_History"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                url: 'https://github.com/bavishasundaram29-lang/jenkins_comparision.git'
            }
        }

        stage('Clean Workspace') {
            steps {
                bat '''
                if exist results rmdir /s /q results
                if exist report rmdir /s /q report
                if exist comparison rmdir /s /q comparison
                if exist zipreport rmdir /s /q zipreport

                mkdir results
                mkdir report
                mkdir comparison
                mkdir zipreport
                '''
            }
        }

        stage('Run JMeter Test') {
            steps {
                bat """
                "%JMETER%" -n ^
                -t "%JMX_FILE%" ^
                -l "results\\SCR01_Build_${BUILD_NUMBER}.jtl" ^
                -e -o "report\\${REPORT_NAME}"
                """
            }
        }

        stage('Generate Current Build Summary') {
            steps {
                script {
                    def stats = readJSON file: "report/${REPORT_NAME}/statistics.json"
                    def total = stats['Total']

                    def currentSummary = [
                        buildNumber : env.BUILD_NUMBER,
                        reportName  : env.REPORT_NAME,
                        sampleCount : total.sampleCount,
                        errorCount  : total.errorCount,
                        errorPct    : total.errorPct,
                        average     : total.meanResTime,
                        min         : total.minResTime,
                        max         : total.maxResTime,
                        median      : total.medianResTime,
                        pct90       : total.pct1ResTime,
                        throughput  : total.throughput
                    ]

                    writeJSON file: 'comparison/current-summary.json',
                    json: currentSummary,
                    pretty: 4
                }
            }
        }

        stage('Compare With Previous Build') {
            steps {
                script {
                    bat """
                    if not exist "%HISTORY_DIR%" mkdir "%HISTORY_DIR%"
                    """

                    def comparisonHtml = """
                    <html>
                    <body>
                    <h2>SCR01 JPetstore Build Comparison Report</h2>
                    <h3>Current Build : #${BUILD_NUMBER}</h3>
                    """

                    if (fileExists("${HISTORY_DIR}\\previous-summary.json")) {

                        bat """
                        copy "%HISTORY_DIR%\\previous-summary.json" "comparison\\previous-summary.json" /Y
                        """

                        def previous = readJSON file: 'comparison/previous-summary.json'
                        def current = readJSON file: 'comparison/current-summary.json'

                        comparisonHtml += """
                        <table border="1" cellpadding="6" cellspacing="0">
                            <tr>
                                <th>Metric</th>
                                <th>Previous Build #${previous.buildNumber}</th>
                                <th>Current Build #${current.buildNumber}</th>
                                <th>Difference</th>
                            </tr>

                            <tr>
                                <td>Samples</td>
                                <td>${previous.sampleCount}</td>
                                <td>${current.sampleCount}</td>
                                <td>${current.sampleCount - previous.sampleCount}</td>
                            </tr>

                            <tr>
                                <td>Error Count</td>
                                <td>${previous.errorCount}</td>
                                <td>${current.errorCount}</td>
                                <td>${current.errorCount - previous.errorCount}</td>
                            </tr>

                            <tr>
                                <td>Error %</td>
                                <td>${String.format("%.2f", previous.errorPct.toDouble())}%</td>
                                <td>${String.format("%.2f", current.errorPct.toDouble())}%</td>
                                <td>${String.format("%.2f", current.errorPct.toDouble() - previous.errorPct.toDouble())}%</td>
                            </tr>

                            <tr>
                                <td>Average Response Time</td>
                                <td>${String.format("%.2f", previous.average.toDouble())} ms</td>
                                <td>${String.format("%.2f", current.average.toDouble())} ms</td>
                                <td>${String.format("%.2f", current.average.toDouble() - previous.average.toDouble())} ms</td>
                            </tr>

                            <tr>
                                <td>Min Response Time</td>
                                <td>${previous.min} ms</td>
                                <td>${current.min} ms</td>
                                <td>${current.min - previous.min} ms</td>
                            </tr>

                            <tr>
                                <td>Max Response Time</td>
                                <td>${previous.max} ms</td>
                                <td>${current.max} ms</td>
                                <td>${current.max - previous.max} ms</td>
                            </tr>

                            <tr>
                                <td>Median Response Time</td>
                                <td>${String.format("%.2f", previous.median.toDouble())} ms</td>
                                <td>${String.format("%.2f", current.median.toDouble())} ms</td>
                                <td>${String.format("%.2f", current.median.toDouble() - previous.median.toDouble())} ms</td>
                            </tr>

                            <tr>
                                <td>90th Percentile</td>
                                <td>${String.format("%.2f", previous.pct90.toDouble())} ms</td>
                                <td>${String.format("%.2f", current.pct90.toDouble())} ms</td>
                                <td>${String.format("%.2f", current.pct90.toDouble() - previous.pct90.toDouble())} ms</td>
                            </tr>

                            <tr>
                                <td>Throughput</td>
                                <td>${String.format("%.2f", previous.throughput.toDouble())}</td>
                                <td>${String.format("%.2f", current.throughput.toDouble())}</td>
                                <td>${String.format("%.2f", current.throughput.toDouble() - previous.throughput.toDouble())}</td>
                            </tr>
                        </table>
                        """

                    } else {
                        comparisonHtml += """
                        <h3>No Previous Build Found</h3>
                        <p>Comparison report will be generated from the second build onwards.</p>
                        """
                    }

                    comparisonHtml += """
                    </body>
                    </html>
                    """

                    writeFile file: 'comparison/comparison-report.html',
                    text: comparisonHtml

                    bat """
                    copy "comparison\\current-summary.json" "%HISTORY_DIR%\\previous-summary.json" /Y
                    """
                }
            }
        }

        stage('Create ZIP Report') {
            steps {
                powershell """
                Compress-Archive -Path "report\\${REPORT_NAME}\\*" -DestinationPath "zipreport\\${REPORT_NAME}.zip" -Force
                Compress-Archive -Path "comparison\\*" -DestinationPath "zipreport\\SCR01_Comparison_Build_${BUILD_NUMBER}.zip" -Force
                """
            }
        }

        stage('Publish Reports In Jenkins UI') {
            steps {
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: "report/${REPORT_NAME}",
                    reportFiles: 'index.html',
                    reportName: "SCR01 HTML Report - Build ${BUILD_NUMBER}"
                ])

                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'comparison',
                    reportFiles: 'comparison-report.html',
                    reportName: "SCR01 Comparison Report - Build ${BUILD_NUMBER}"
                ])
            }
        }
    }

    post {
        always {
            emailext(
                subject: "Jenkins SCR01 Comparison Report - Build ${BUILD_NUMBER}",
                mimeType: 'text/html',
                to: 'bavishasundar@gmail.com',
                body: """
                <html>
                <body>
                <h2>SCR01 JPetstore Execution Completed</h2>
                <h3>Job Name : Jenkins_Comparision</h3>
                <h3>Build Number : ${BUILD_NUMBER}</h3>
                <h3>Build Status : ${currentBuild.currentResult}</h3>

                <p>Reports are published in Jenkins UI and attached as ZIP files.</p>

                <ul>
                    <li>SCR01 HTML Report - Build ${BUILD_NUMBER}</li>
                    <li>SCR01 Comparison Report - Build ${BUILD_NUMBER}</li>
                </ul>

                <a href="${BUILD_URL}">Open Jenkins Build</a>
                </body>
                </html>
                """,
                attachmentsPattern: 'zipreport/*.zip'
            )

            archiveArtifacts artifacts: 'results/*.jtl, report/**/*, comparison/**/*, zipreport/*.zip',
            fingerprint: true
        }
    }
}
