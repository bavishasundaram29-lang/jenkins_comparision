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

        stage('Generate Current Summary') {
            steps {
                script {

                    def stats = readJSON file: "report/${REPORT_NAME}/statistics.json"

                    def apiSummary = [:]

                    stats.each { apiName, data ->

                        if (apiName != "Total") {

                            apiSummary[apiName] = [
                                responseTime : data.meanResTime ?: 0,
                                samples      : data.sampleCount ?: 0,
                                errors       : data.errorCount ?: 0
                            ]
                        }
                    }

                    def currentSummary = [
                        buildNumber : env.BUILD_NUMBER,
                        apis        : apiSummary
                    ]

                    writeJSON file: 'aggregate-report/current-summary.json',
                    json: currentSummary,
                    pretty: 4
                }
            }
        }

        stage('Create API Wise Comparison Report') {
            steps {
                script {

                    bat """
                    if not exist "%HISTORY_DIR%" mkdir "%HISTORY_DIR%"
                    """

                    def html = """
                    <html>

                    <head>

                    <style>

                        body {
                            font-family: Arial;
                            margin: 20px;
                        }

                        h1 {
                            color: black;
                        }

                        h2 {
                            color: black;
                        }

                        table {
                            border-collapse: collapse;
                            width: 100%;
                            margin-top: 20px;
                        }

                        th {
                            border: 1px solid black;
                            padding: 8px;
                            text-align: center;
                            background-color: #f2f2f2;
                            font-weight: bold;
                        }

                        td {
                            border: 1px solid black;
                            padding: 8px;
                            text-align: center;
                        }

                    </style>

                    </head>

                    <body>

                    <h1>API Wise Comparison Report</h1>

                    """

                    if (fileExists("${HISTORY_DIR}\\previous-summary.json")) {

                        bat """
                        copy "%HISTORY_DIR%\\previous-summary.json" "aggregate-report\\previous-summary.json" /Y
                        """

                        def previous = readJSON file: 'aggregate-report/previous-summary.json'

                        def current = readJSON file: 'aggregate-report/current-summary.json'

                        html += """
                        <h2>Build #${previous.buildNumber} vs Build #${current.buildNumber}</h2>

                        <table>

                            <tr>
                                <th rowspan="2">API Name</th>

                                <th colspan="4">Response Time (ms)</th>

                                <th colspan="3">Samples</th>

                                <th colspan="3">Errors</th>
                            </tr>

                            <tr>

                                <th>Previous</th>
                                <th>Current</th>
                                <th>Deviation</th>
                                <th>%</th>

                                <th>Previous</th>
                                <th>Current</th>
                                <th>Deviation</th>

                                <th>Previous</th>
                                <th>Current</th>
                                <th>Deviation</th>

                            </tr>
                        """

                        current.apis.each { apiName, cur ->

                            def prev = previous.apis[apiName] ?: [
                                responseTime : 0,
                                samples : 0,
                                errors : 0
                            ]

                            double prevRT = (prev.responseTime ?: 0) as double
                            double curRT  = (cur.responseTime ?: 0) as double

                            double rtDev = curRT - prevRT

                            double rtPct = prevRT > 0 ? (rtDev / prevRT) * 100 : 0

                            int prevSamples = (prev.samples ?: 0) as int
                            int curSamples  = (cur.samples ?: 0) as int

                            int sampleDev = curSamples - prevSamples

                            int prevErrors = (prev.errors ?: 0) as int
                            int curErrors  = (cur.errors ?: 0) as int

                            int errorDev = curErrors - prevErrors

                            html += """
                            <tr>

                                <td>${apiName}</td>

                                <td>${String.format("%.4f", prevRT)}</td>

                                <td>${String.format("%.4f", curRT)}</td>

                                <td>${String.format("%.4f", rtDev)}</td>

                                <td>${String.format("%.2f", rtPct)}%</td>

                                <td>${prevSamples}</td>

                                <td>${curSamples}</td>

                                <td>${sampleDev}</td>

                                <td>${prevErrors}</td>

                                <td>${curErrors}</td>

                                <td>${errorDev}</td>

                            </tr>
                            """
                        }

                        html += """
                        </table>
                        """

                    }
                    else {

                        html += """
                        <h2>No Previous Build Found</h2>

                        <p>
                        Comparison report will generate from next successful build onwards.
                        </p>
                        """
                    }

                    html += """
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
                    reportName: "SCR01 Aggregate Comparison Report"
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

                <p>
                Aggregate comparison report is generated successfully.
                </p>

                <a href="${BUILD_URL}">
                Open Jenkins Build
                </a>

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
