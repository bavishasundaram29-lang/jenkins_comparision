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
                if exist aggregate-report rmdir /s /q aggregate-report
                if exist zipreport rmdir /s /q zipreport

                mkdir results
                mkdir report
                mkdir aggregate-report
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

        stage('Generate Current Aggregate Summary') {
            steps {
                script {
                    def stats = readJSON file: "report/${REPORT_NAME}/statistics.json"
                    def total = stats['Total']

                    def currentSummary = [
                        buildNumber : env.BUILD_NUMBER,
                        samples     : total.sampleCount ?: 0,
                        failures    : total.errorCount ?: 0,
                        errorPct    : total.errorPct ?: 0,
                        average     : total.meanResTime ?: 0,
                        min         : total.minResTime ?: 0,
                        max         : total.maxResTime ?: 0,
                        median      : total.medianResTime ?: 0,
                        pct90       : total.pct1ResTime ?: 0,
                        pct95       : total.pct2ResTime ?: 0,
                        pct99       : total.pct3ResTime ?: 0,
                        throughput  : total.throughput ?: 0
                    ]

                    writeJSON file: 'aggregate-report/current-summary.json',
                    json: currentSummary,
                    pretty: 4
                }
            }
        }

        stage('Create Aggregate Comparison Report') {
            steps {
                script {
                    bat """
                    if not exist "%HISTORY_DIR%" mkdir "%HISTORY_DIR%"
                    """

                    def html = """
                    <html>
                    <body>
                    <h2>SCR01 JPetstore Aggregate Comparison Report</h2>
                    <h3>Job Name : Jenkins_Comparision</h3>
                    <h3>Current Build : #${BUILD_NUMBER}</h3>
                    <br>
                    """

                    if (fileExists("${HISTORY_DIR}\\previous-summary.json")) {

                        bat """
                        copy "%HISTORY_DIR%\\previous-summary.json" "aggregate-report\\previous-summary.json" /Y
                        """

                        def previous = readJSON file: 'aggregate-report/previous-summary.json'
                        def current = readJSON file: 'aggregate-report/current-summary.json'

                        html += """
                        <table border="1" cellpadding="6" cellspacing="0">
                            <tr>
                                <th>Metric</th>
                                <th>Previous Build #${previous.buildNumber}</th>
                                <th>Current Build #${current.buildNumber}</th>
                                <th>Difference</th>
                            </tr>

                            <tr>
                                <td>Total Samples</td>
                                <td>${previous.samples ?: 0}</td>
                                <td>${current.samples ?: 0}</td>
                                <td>${(current.samples ?: 0) - (previous.samples ?: 0)}</td>
                            </tr>

                            <tr>
                                <td>Failures</td>
                                <td>${previous.failures ?: 0}</td>
                                <td>${current.failures ?: 0}</td>
                                <td>${(current.failures ?: 0) - (previous.failures ?: 0)}</td>
                            </tr>

                            <tr>
                                <td>Error %</td>
                                <td>${String.format("%.2f", (previous.errorPct ?: 0) * 1.0)}%</td>
                                <td>${String.format("%.2f", (current.errorPct ?: 0) * 1.0)}%</td>
                                <td>${String.format("%.2f", ((current.errorPct ?: 0) * 1.0) - ((previous.errorPct ?: 0) * 1.0))}%</td>
                            </tr>

                            <tr>
                                <td>Average Response Time</td>
                                <td>${String.format("%.2f", (previous.average ?: 0) * 1.0)} ms</td>
                                <td>${String.format("%.2f", (current.average ?: 0) * 1.0)} ms</td>
                                <td>${String.format("%.2f", ((current.average ?: 0) * 1.0) - ((previous.average ?: 0) * 1.0))} ms</td>
                            </tr>

                            <tr>
                                <td>Minimum Response Time</td>
                                <td>${previous.min ?: 0} ms</td>
                                <td>${current.min ?: 0} ms</td>
                                <td>${(current.min ?: 0) - (previous.min ?: 0)} ms</td>
                            </tr>

                            <tr>
                                <td>Maximum Response Time</td>
                                <td>${previous.max ?: 0} ms</td>
                                <td>${current.max ?: 0} ms</td>
                                <td>${(current.max ?: 0) - (previous.max ?: 0)} ms</td>
                            </tr>

                            <tr>
                                <td>Median Response Time</td>
                                <td>${String.format("%.2f", (previous.median ?: 0) * 1.0)} ms</td>
                                <td>${String.format("%.2f", (current.median ?: 0) * 1.0)} ms</td>
                                <td>${String.format("%.2f", ((current.median ?: 0) * 1.0) - ((previous.median ?: 0) * 1.0))} ms</td>
                            </tr>

                            <tr>
                                <td>90th Percentile</td>
                                <td>${String.format("%.2f", (previous.pct90 ?: 0) * 1.0)} ms</td>
                                <td>${String.format("%.2f", (current.pct90 ?: 0) * 1.0)} ms</td>
                                <td>${String.format("%.2f", ((current.pct90 ?: 0) * 1.0) - ((previous.pct90 ?: 0) * 1.0))} ms</td>
                            </tr>

                            <tr>
                                <td>95th Percentile</td>
                                <td>${String.format("%.2f", (previous.pct95 ?: 0) * 1.0)} ms</td>
                                <td>${String.format("%.2f", (current.pct95 ?: 0) * 1.0)} ms</td>
                                <td>${String.format("%.2f", ((current.pct95 ?: 0) * 1.0) - ((previous.pct95 ?: 0) * 1.0))} ms</td>
                            </tr>

                            <tr>
                                <td>99th Percentile</td>
                                <td>${String.format("%.2f", (previous.pct99 ?: 0) * 1.0)} ms</td>
                                <td>${String.format("%.2f", (current.pct99 ?: 0) * 1.0)} ms</td>
                                <td>${String.format("%.2f", ((current.pct99 ?: 0) * 1.0) - ((previous.pct99 ?: 0) * 1.0))} ms</td>
                            </tr>

                            <tr>
                                <td>Throughput</td>
                                <td>${String.format("%.2f", (previous.throughput ?: 0) * 1.0)}</td>
                                <td>${String.format("%.2f", (current.throughput ?: 0) * 1.0)}</td>
                                <td>${String.format("%.2f", ((current.throughput ?: 0) * 1.0) - ((previous.throughput ?: 0) * 1.0))}</td>
                            </tr>
                        </table>
                        """

                    } else {
                        html += """
                        <h3>No Previous Build Found</h3>
                        <p>From the next successful build, comparison will be generated.</p>
                        """
                    }

                    html += """
                    <br>
                    <h3>Workspace Report Path</h3>
                    <p>aggregate-report\\aggregate-comparison-report.html</p>
                    </body>
                    </html>
                    """

                    writeFile file: 'aggregate-report/aggregate-comparison-report.html',
                    text: html

                    bat """
                    copy "aggregate-report\\current-summary.json" "%HISTORY_DIR%\\previous-summary.json" /Y
                    """
                }
            }
        }

        stage('Create ZIP Report') {
            steps {
                powershell """
                Compress-Archive -Path "aggregate-report\\*" -DestinationPath "zipreport\\SCR01_Aggregate_Comparison_Build_${BUILD_NUMBER}.zip" -Force
                """
            }
        }

        stage('Publish Aggregate Report In Jenkins UI') {
            steps {
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'aggregate-report',
                    reportFiles: 'aggregate-comparison-report.html',
                    reportName: "SCR01 Aggregate Comparison Report - Build ${BUILD_NUMBER}"
                ])
            }
        }
    }

    post {
        always {
            emailext(
                subject: "SCR01 Aggregate Comparison Report - Build ${BUILD_NUMBER}",
                mimeType: 'text/html',
                to: 'bavishasundar@gmail.com',
                body: """
                <html>
                <body>
                <h2>SCR01 Aggregate Comparison Report Generated</h2>
                <h3>Job Name : Jenkins_Comparision</h3>
                <h3>Build Number : ${BUILD_NUMBER}</h3>
                <h3>Build Status : ${currentBuild.currentResult}</h3>

                <p>The aggregate comparison report is available in Jenkins UI, workspace, and email attachment.</p>

                <a href="${BUILD_URL}">Open Jenkins Build</a>
                </body>
                </html>
                """,
                attachmentsPattern: 'zipreport/*.zip'
            )

            archiveArtifacts artifacts: 'results/*.jtl, report/**/*, aggregate-report/**/*, zipreport/*.zip',
            fingerprint: true
        }
    }
}
