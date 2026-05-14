pipeline {
    agent any

    parameters {
        string(name: 'BUILD_1', defaultValue: '', description: 'Enter first build number to compare')
        string(name: 'BUILD_2', defaultValue: '', description: 'Enter second build number to compare')
    }

    environment {
        JMETER_HOME = "C:\\apache-jmeter-5.6.3\\apache-jmeter-5.6.3"
        JMETER = "${JMETER_HOME}\\bin\\jmeter.bat"
        JMX_FILE = "jpetstore_jenkins_comparision\\SCR01_Jpetstore.jmx"

        HISTORY_DIR = "C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\Jenkins_Comparision_History"

        REPORT_NAME = "SCR01_Report_Build_${BUILD_NUMBER}"
        ZIP_NAME = "SCR01_Script_Comparison_Build_${BUILD_NUMBER}.zip"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                url: 'https://github.com/bavishasundaram29-lang/jenkins_comparision.git'
            }
        }

        stage('Clean Current Workspace') {
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

        stage('Save Current Build Report History') {
            steps {
                script {
                    bat """
                    if not exist "%HISTORY_DIR%" mkdir "%HISTORY_DIR%"
                    if not exist "%HISTORY_DIR%\\Build_${BUILD_NUMBER}" mkdir "%HISTORY_DIR%\\Build_${BUILD_NUMBER}"
                    """

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
                        file: "aggregate-report/Build_${BUILD_NUMBER}_summary.json",
                        json: [
                            buildNumber : env.BUILD_NUMBER,
                            apis        : apiSummary
                        ],
                        pretty: 4
                    )

                    bat """
                    copy "aggregate-report\\Build_${BUILD_NUMBER}_summary.json" "%HISTORY_DIR%\\Build_${BUILD_NUMBER}_summary.json" /Y
                    copy "aggregate-report\\Build_${BUILD_NUMBER}_summary.json" "%HISTORY_DIR%\\Build_${BUILD_NUMBER}\\Build_${BUILD_NUMBER}_summary.json" /Y

                    xcopy "results" "%HISTORY_DIR%\\Build_${BUILD_NUMBER}\\results\\" /E /I /Y
                    xcopy "report" "%HISTORY_DIR%\\Build_${BUILD_NUMBER}\\report\\" /E /I /Y
                    """
                }
            }
        }

        stage('Create Selected Build Comparison Report') {
            steps {
                script {
                    def build1 = params.BUILD_1?.trim()
                    def build2 = params.BUILD_2?.trim()

                    if (build1 == "" || build2 == "") {
                        echo "BUILD_1 or BUILD_2 is empty. Skipping comparison report."
                        writeFile file: 'aggregate-report/aggregate-comparison-report.html',
                        text: """
                        <html>
                        <body>
                        <h1>Script Wise Comparison Report</h1>
                        <h2>No Build Numbers Given</h2>
                        <p>Current build report and summary are stored successfully.</p>
                        </body>
                        </html>
                        """
                        return
                    }

                    def build1File = "${HISTORY_DIR}\\Build_${build1}_summary.json"
                    def build2File = "${HISTORY_DIR}\\Build_${build2}_summary.json"

                    if (!fileExists(build1File)) {
                        error "Build ${build1} summary not found in ${HISTORY_DIR}"
                    }

                    if (!fileExists(build2File)) {
                        error "Build ${build2} summary not found in ${HISTORY_DIR}"
                    }

                    bat """
                    copy "${build1File}" "aggregate-report\\build1-summary.json" /Y
                    copy "${build2File}" "aggregate-report\\build2-summary.json" /Y
                    """

                    def previous = readJSON file: 'aggregate-report/build1-summary.json'
                    def current = readJSON file: 'aggregate-report/build2-summary.json'

                    def html = """
                    <html>
                    <head>
                    <style>
                        body { font-family: Arial; margin: 20px; }
                        h1, h2 { color: black; }
                        table { border-collapse: collapse; width: 100%; margin-top: 20px; }
                        th, td { border: 1px solid black; padding: 8px; }
                        th { background-color: #f2f2f2; text-align: center; font-weight: bold; }
                        td { text-align: center; }
                        .transaction-col { text-align: left; font-weight: bold; white-space: nowrap; }
                        .request-col { text-align: left; word-break: break-word; }
                    </style>
                    </head>
                    <body>

                    <h1>Script Wise Comparison Report</h1>
                    <h2>Build #${build1} vs Build #${build2}</h2>

                    <table>
                        <tr>
                            <th rowspan="2">Transaction</th>
                            <th rowspan="2">Request</th>
                            <th colspan="4">Response Time (ms)</th>
                            <th colspan="3">Samples</th>
                            <th colspan="3">Errors</th>
                        </tr>

                        <tr>
                            <th>Build #${build1}</th>
                            <th>Build #${build2}</th>
                            <th>Deviation</th>
                            <th>%</th>

                            <th>Build #${build1}</th>
                            <th>Build #${build2}</th>
                            <th>Deviation</th>

                            <th>Build #${build1}</th>
                            <th>Build #${build2}</th>
                            <th>Deviation</th>
                        </tr>
                    """

                    def transactionMap = [:]
                    def transactionNameMap = [:]

                    current.apis.keySet().toList().sort().each { api ->
                        String apiName = api.toString()

                        def matcher = apiName =~ /(SCR[0-9]+_T[0-9]+)/
                        String transactionKey = "OTHERS"

                        if (matcher.find()) {
                            transactionKey = matcher.group(1)
                        }

                        if (!transactionMap.containsKey(transactionKey)) {
                            transactionMap[transactionKey] = []
                        }

                        transactionMap[transactionKey] << apiName

                        if (!(apiName =~ /_R[0-9]+/)) {
                            transactionNameMap[transactionKey] = apiName.replaceAll('_', ' ')
                        }
                    }

                    transactionMap.keySet().toList().sort().each { transactionKey ->

                        def apiList = transactionMap[transactionKey].toList().sort()
                        def transactionRows = apiList.findAll { !(it =~ /_R[0-9]+/) }
                        def requestRows = apiList.findAll { it =~ /_R[0-9]+/ }

                        def orderedList = []
                        orderedList.addAll(transactionRows)
                        orderedList.addAll(requestRows)

                        String transactionDisplay = transactionNameMap[transactionKey]

                        if (transactionDisplay == null || transactionDisplay.trim() == "") {
                            transactionDisplay = transactionKey.replaceAll('_', ' ')
                        }

                        orderedList.each { apiName ->

                            def prev = previous.apis[apiName]
                            def cur = current.apis[apiName]

                            if (prev == null) {
                                prev = [responseTime: 0, samples: 0, errors: 0]
                            }

                            if (cur == null) {
                                cur = [responseTime: 0, samples: 0, errors: 0]
                            }

                            String requestName = apiName.replaceAll('_', ' ')

                            double prevRT = (prev.responseTime ?: 0) as double
                            double curRT  = (cur.responseTime ?: 0) as double
                            double rtDev  = curRT - prevRT
                            double rtPct  = prevRT > 0 ? ((rtDev / prevRT) * 100) : 0

                            int prevSamples = (prev.samples ?: 0) as int
                            int curSamples  = (cur.samples ?: 0) as int
                            int sampleDev   = curSamples - prevSamples

                            int prevErrors = (prev.errors ?: 0) as int
                            int curErrors  = (cur.errors ?: 0) as int
                            int errorDev   = curErrors - prevErrors

                            html += """
                            <tr>
                                <td class="transaction-col">${transactionDisplay}</td>
                                <td class="request-col">${requestName}</td>

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
                    </body>
                    </html>
                    """

                    writeFile file: 'aggregate-report/aggregate-comparison-report.html', text: html

                    bat """
                    copy "aggregate-report\\aggregate-comparison-report.html" "%HISTORY_DIR%\\Build_${BUILD_NUMBER}\\aggregate-comparison-report.html" /Y
                    """
                }
            }
        }

        stage('Create ZIP Report') {
            steps {
                powershell """
                Compress-Archive -Path "aggregate-report\\*" -DestinationPath "zipreport\\${ZIP_NAME}" -Force
                """
                bat """
                copy "zipreport\\${ZIP_NAME}" "%HISTORY_DIR%\\Build_${BUILD_NUMBER}\\${ZIP_NAME}" /Y
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

                <h3>Current Jenkins Build : #${BUILD_NUMBER}</h3>
                <h3>Compared Builds : #${params.BUILD_1} vs #${params.BUILD_2}</h3>
                <h3>Build Status : ${currentBuild.currentResult}</h3>

                <p>All build reports are stored in:</p>
                <p><b>${HISTORY_DIR}</b></p>

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
