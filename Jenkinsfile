pipeline {
    agent any

    environment {
        JMETER_HOME = "C:\\apache-jmeter-5.6.3\\apache-jmeter-5.6.3"
        JMETER = "${JMETER_HOME}\\bin\\jmeter.bat"
        JMX_FILE = "jpetstore_jenkins_comparision\\SCR01_Jpetstore.jmx"
        REPORT_NAME = "SCR01_Report_Build_${BUILD_NUMBER}"
        HISTORY_DIR = "C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\Jenkins_Comparision_History"
        ZIP_NAME = "SCR01_Script_Comparison_Build_${BUILD_NUMBER}.zip"
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

                            apiSummary[apiName.toString()] = [
                                responseTime : data.meanResTime ?: 0,
                                samples      : data.sampleCount ?: 0,
                                errors       : data.errorCount ?: 0
                            ]
                        }
                    }

                    writeJSON(
                        file: 'aggregate-report/current-summary.json',
                        json: [
                            buildNumber : env.BUILD_NUMBER,
                            apis        : apiSummary
                        ],
                        pretty: 4
                    )
                }
            }
        }

        stage('Create Script Wise Comparison Report') {
            steps {
                script {

                    bat """
                    if not exist "%HISTORY_DIR%" mkdir "%HISTORY_DIR%"
                    """

                    def current = readJSON file: 'aggregate-report/current-summary.json'

                    def previous = null

                    if (fileExists("${HISTORY_DIR}\\previous-summary.json")) {

                        bat """
                        copy "%HISTORY_DIR%\\previous-summary.json" "aggregate-report\\previous-summary.json" /Y
                        """

                        previous = readJSON file: 'aggregate-report/previous-summary.json'
                    }

                    def html = """
                    <html>

                    <head>

                    <style>

                        body {
                            font-family: Arial;
                            margin: 20px;
                        }

                        h1, h2 {
                            color: black;
                        }

                        table {
                            border-collapse: collapse;
                            width: 100%;
                            margin-top: 20px;
                        }

                        th, td {
                            border: 1px solid black;
                            padding: 8px;
                        }

                        th {
                            background-color: #f2f2f2;
                            font-weight: bold;
                            text-align: center;
                        }

                        td {
                            text-align: center;
                        }

                        .transaction-col {
                            text-align: left;
                            padding-left: 12px;
                            font-weight: bold;
                            white-space: nowrap;
                        }

                        .request-col {
                            text-align: left;
                            padding-left: 12px;
                            word-break: break-word;
                        }

                    </style>

                    </head>

                    <body>

                    <h1>Script Wise Comparison Report</h1>
                    """

                    if (previous != null && previous.apis != null) {

                        html += """
                        <h2>Build #${previous.buildNumber} vs Build #${current.buildNumber}</h2>

                        <table>

                            <tr>

                                <th rowspan="2">Transaction</th>

                                <th rowspan="2">Request</th>

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

                        def sortedApis = current.apis.keySet().toList().sort()

                        def transactionMap = [:]

                        sortedApis.each { api ->

                            String apiName = api.toString()

                            def transactionKey = "OTHERS"

                            def tMatch = (apiName =~ /(SCR\\d+_T\\d+)/)

                            if (tMatch.find()) {
                                transactionKey = tMatch.group(1)
                            }

                            if (!transactionMap.containsKey(transactionKey)) {
                                transactionMap[transactionKey] = []
                            }

                            transactionMap[transactionKey] << apiName
                        }

                        transactionMap.each { transactionKey, apiList ->

                            String transactionDisplay = transactionKey.replaceAll('_', ' ')

                            def launchRequest = apiList.find {
                                !(it.contains("/"))
                            }

                            if (launchRequest == null) {
                                launchRequest = apiList[0]
                            }

                            apiList.each { apiName ->

                                def cur = current.apis[apiName]

                                def prev = previous.apis[apiName]

                                if (prev == null) {

                                    prev = [
                                        responseTime : 0,
                                        samples      : 0,
                                        errors       : 0
                                    ]
                                }

                                String requestName = apiName

                                requestName = requestName
                                    .replaceAll('_', ' ')
                                    .trim()

                                double prevRT = (prev.responseTime ?: 0) as double
                                double curRT  = (cur.responseTime ?: 0) as double

                                double rtDev = curRT - prevRT

                                double rtPct = prevRT > 0 ? ((rtDev / prevRT) * 100) : 0

                                int prevSamples = (prev.samples ?: 0) as int
                                int curSamples  = (cur.samples ?: 0) as int

                                int sampleDev = curSamples - prevSamples

                                int prevErrors = (prev.errors ?: 0) as int
                                int curErrors  = (cur.errors ?: 0) as int

                                int errorDev = curErrors - prevErrors

                                html += """
                                <tr>

                                    <td class="transaction-col">
                                    ${transactionDisplay}
                                    </td>

                                    <td class="request-col">
                                    ${requestName}
                                    </td>

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
                        }

                        html += """
                        </table>
                        """

                    } else {

                        html += """
                        <h2>No Previous Build Found</h2>

                        <p>
                        Current build summary saved successfully.
                        Comparison report will generate from next build.
                        </p>
                        """
                    }

                    html += """
                    </body>
                    </html>
                    """

                    writeFile(
                        file: 'aggregate-report/aggregate-comparison-report.html',
                        text: html
                    )

                    bat """
                    copy "aggregate-report\\current-summary.json" "%HISTORY_DIR%\\previous-summary.json" /Y
                    """
                }
            }
        }

        stage('Create ZIP Report') {
            steps {

                powershell """
                Compress-Archive -Path "aggregate-report\\*" -DestinationPath "zipreport\\${ZIP_NAME}" -Force
                """
            }
        }

        stage('Publish Report In Jenkins UI') {
            steps {

                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'aggregate-report',
                    reportFiles: 'aggregate-comparison-report.html',
                    reportName: 'Script Wise Comparison Report'
                ])
            }
        }

        stage('Archive Reports') {
            steps {

                archiveArtifacts(
                    artifacts: 'results/*.jtl, report/**/*, aggregate-report/**/*, zipreport/*.zip',
                    fingerprint: true
                )
            }
        }
    }

    post {

        always {

            emailext(

                subject: "SCR01 Script Wise Comparison Report - Build ${BUILD_NUMBER}",

                mimeType: 'text/html',

                to: 'bavishasundar@gmail.com',

                body: """
                <html>

                <body>

                <h2>SCR01 Script Wise Comparison Report Generated</h2>

                <h3>Job Name : Jenkins_Comparision</h3>

                <h3>Build Number : ${BUILD_NUMBER}</h3>

                <h3>Build Status : ${currentBuild.currentResult}</h3>

                <p>
                Script wise comparison report generated successfully.
                </p>

                <p>
                <b>Download ZIP Report:</b>

                <a href="${BUILD_URL}artifact/zipreport/${ZIP_NAME}">
                Click here to download ZIP
                </a>
                </p>

                <p>
                <b>Open Jenkins Build:</b>

                <a href="${BUILD_URL}">
                Click here
                </a>
                </p>

                </body>

                </html>
                """,

                attachmentsPattern: 'zipreport/*.zip, aggregate-report/aggregate-comparison-report.html'
            )
        }
    }
}
