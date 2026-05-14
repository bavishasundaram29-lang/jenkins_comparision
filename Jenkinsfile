pipeline {
    agent any

    parameters {
        string(name: 'BUILD_1', defaultValue: '10', description: 'Enter first build number')
        string(name: 'BUILD_2', defaultValue: '11', description: 'Enter second build number')
    }

    environment {
        JMETER_HOME = "C:\\apache-jmeter-5.6.3\\apache-jmeter-5.6.3"
        JMETER = "${JMETER_HOME}\\bin\\jmeter.bat"
        JMX_FILE = "jpetstore_jenkins_comparision\\SCR01_Jpetstore.jmx"
        REPORT_NAME = "SCR01_Report_Build_${BUILD_NUMBER}"
        HISTORY_DIR = "C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\Jenkins_Comparision_History"
        ZIP_NAME = "SCR01_Custom_Comparison_Build_${BUILD_1}_vs_${BUILD_2}.zip"
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

        stage('Run Current JMeter Test') {
            steps {
                bat """
                "%JMETER%" -n ^
                -t "%JMX_FILE%" ^
                -l "results\\SCR01_Build_${BUILD_NUMBER}.jtl" ^
                -e -o "report\\${REPORT_NAME}"
                """
            }
        }

        stage('Save Current Build Summary') {
            steps {
                script {
                    bat """
                    if not exist "%HISTORY_DIR%" mkdir "%HISTORY_DIR%"
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
                    """
                }
            }
        }

        stage('Create Selected Build Comparison Report') {
            steps {
                script {
                    def build1File = "${HISTORY_DIR}\\Build_${params.BUILD_1}_summary.json"
                    def build2File = "${HISTORY_DIR}\\Build_${params.BUILD_2}_summary.json"

                    if (!fileExists(build1File)) {
                        error "Build ${params.BUILD_1} summary not found. Run build ${params.BUILD_1} first using this Jenkinsfile."
                    }

                    if (!fileExists(build2File)) {
                        error "Build ${params.BUILD_2} summary not found. Run build ${params.BUILD_2} first using this Jenkinsfile."
                    }

                    bat """
                    copy "${build1File}" "aggregate-report\\build1-summary.json" /Y
                    copy "${build2File}" "aggregate-report\\build2-summary.json" /Y
                    """

                    def build1 = readJSON file: 'aggregate-report/build1-summary.json'
                    def build2 = readJSON file: 'aggregate-report/build2-summary.json'

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
                    <h2>Build #${params.BUILD_1} vs Build #${params.BUILD_2}</h2>

                    <table>
                        <tr>
                            <th rowspan="2">Transaction</th>
                            <th rowspan="2">Request</th>
                            <th colspan="4">Response Time (ms)</th>
                            <th colspan="3">Samples</th>
                            <th colspan="3">Errors</th>
                        </tr>

                        <tr>
                            <th>Build #${params.BUILD_1}</th>
                            <th>Build #${params.BUILD_2}</th>
                            <th>Deviation</th>
                            <th>%</th>

                            <th>Build #${params.BUILD_1}</th>
                            <th>Build #${params.BUILD_2}</th>
                            <th>Deviation</th>

                            <th>Build #${params.BUILD_1}</th>
                            <th>Build #${params.BUILD_2}</th>
                            <th>Deviation</th>
                        </tr>
                    """

                    def transactionMap = [:]
                    def transactionNameMap = [:]

                    build2.apis.keySet().toList().sort().each { api ->
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

                            def b1 = build1.apis[apiName]
                            def b2 = build2.apis[apiName]

                            if (b1 == null) {
                                b1 = [responseTime: 0, samples: 0, errors: 0]
                            }

                            if (b2 == null) {
                                b2 = [responseTime: 0, samples: 0, errors: 0]
                            }

                            String requestName = apiName.replaceAll('_', ' ')

                            double b1RT = (b1.responseTime ?: 0) as double
                            double b2RT = (b2.responseTime ?: 0) as double
                            double rtDev = b2RT - b1RT
                            double rtPct = b1RT > 0 ? ((rtDev / b1RT) * 100) : 0

                            int b1Samples = (b1.samples ?: 0) as int
                            int b2Samples = (b2.samples ?: 0) as int
                            int sampleDev = b2Samples - b1Samples

                            int b1Errors = (b1.errors ?: 0) as int
                            int b2Errors = (b2.errors ?: 0) as int
                            int errorDev = b2Errors - b1Errors

                            html += """
                            <tr>
                                <td class="transaction-col">${transactionDisplay}</td>
                                <td class="request-col">${requestName}</td>

                                <td>${String.format("%.4f", b1RT)}</td>
                                <td>${String.format("%.4f", b2RT)}</td>
                                <td>${String.format("%.4f", rtDev)}</td>
                                <td>${String.format("%.2f", rtPct)}%</td>

                                <td>${b1Samples}</td>
                                <td>${b2Samples}</td>
                                <td>${sampleDev}</td>

                                <td>${b1Errors}</td>
                                <td>${b2Errors}</td>
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

                    writeFile(
                        file: 'aggregate-report/aggregate-comparison-report.html',
                        text: html
                    )
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
                subject: "SCR01 Script Wise Comparison Report - Build ${params.BUILD_1} vs ${params.BUILD_2}",
                mimeType: 'text/html',
                to: 'bavishasundar@gmail.com',
                body: """
                <html>
                <body>

                <h2>SCR01 Script Wise Comparison Report Generated</h2>

                <h3>Compared Builds : #${params.BUILD_1} vs #${params.BUILD_2}</h3>
                <h3>Build Status : ${currentBuild.currentResult}</h3>

                <p>Selected build comparison report generated successfully.</p>

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
