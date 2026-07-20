pipeline {
    agent any

    environment {
        SONAR_HOST_URL = 'http://192.168.1.158:9000'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Build & Sonar Scan All Projects') {
            steps {
                script {

                    def projects = [
                        [
                            name: 'Facheck',
                            repo: 'https://github.com/Baabujiventuress/Facheck_Springboot_Backend',
                            branch: 'Facheck-Dev-DailyBuild',
                            credentials: 'abhishek',
                            sonarKey: 'Facheck-Dev-Backend',
                            sonarName: 'Facheck-Dev-Backend',
                            mail: 'abhishek.p@babujiventures.in'
                        ]
                    ]

                    def htmlRows = ""
                    def csvContent = "Project,Severity,Type,File,Line,Message,Status\n"
                    projects.each { project ->

                        echo "================================================="
                        echo "Processing ${project.name}"
                        echo "================================================="
                        def projectCsvContent =
                                    "Project,Severity,Type,File,Line,Message\n"
                        
                        try {

                            dir(project.name) {

                                deleteDir()

                                git(
                                    branch: project.branch,
                                    credentialsId: project.credentials,
                                    url: project.repo
                                )

                                sh 'mvn clean'
                                sh 'mvn install'

                                withSonarQubeEnv('sonarqube') {
                                    sh """
                                        mvn sonar:sonar \
                                        -Dsonar.projectKey=${project.sonarKey} \
                                        -Dsonar.projectName=${project.sonarName}
                                    """
                                }

                                timeout(time: 30, unit: 'MINUTES') {

                                    def qg = waitForQualityGate(abortPipeline: false)
                                    def qualityGate = qg.status

                                    withCredentials([
                                        string(
                                            credentialsId: 'sonar-token',
                                            variable: 'SONAR_TOKEN'
                                        )
                                    ]) {

                                        def response = sh(
                                            script: """
                                                curl -s -u \$SONAR_TOKEN: \
                                                "${SONAR_HOST_URL}/api/measures/component?component=${project.sonarKey}&metricKeys=vulnerabilities,bugs,code_smells,duplicated_lines_density,security_hotspots"
                                            """,
                                            returnStdout: true
                                        ).trim()
                                        
                                        echo "Sonar Response: ${response}"
                                        
                                        def json = readJSON text: response
                                        
                                        /* ------------------- GET SONAR ISSUES ------------------- */
                                        
                                        def issuesResponse = sh(
                                            script: """
                                                curl -s -u \$SONAR_TOKEN: \
                                                "${SONAR_HOST_URL}/api/issues/search?componentKeys=${project.sonarKey}&types=BUG,VULNERABILITY&resolved=false&ps=10000"
                                            """,
                                            returnStdout: true
                                        ).trim()
                                        
                                        echo "Issues Response Length: ${issuesResponse.length()}"

                                        def issuesJson = readJSON text: issuesResponse
                                        
                                        echo "Issues Response Full = ${issuesResponse}"
                                        
                                        if (issuesJson?.issues) {
                                        
                                            echo "Total Issues Returned: ${issuesJson.issues.size()}"
                                        
                                            issuesJson.issues.take(10).each { issue ->
                                                echo "TYPE=${issue.type} | SEVERITY=${issue.severity} | MESSAGE=${issue.message}"
                                            }
                                        
                                        } else {
                                        
                                            echo "No issues returned from SonarQube"
                                        
                                        }
                                        
                                        if (issuesJson?.issues) {
                                        
                                            issuesJson.issues.each { issue ->
                                        
                                                if (
                                                    issue.type == 'BUG' ||
                                                    issue.type == 'VULNERABILITY'
                                                ) {
                                        
                                                    projectCsvContent += "\"${project.name}\","
                                                    projectCsvContent += "\"${issue.severity ?: ''}\","
                                                    projectCsvContent += "\"${issue.type ?: ''}\","
                                                    projectCsvContent += "\"${issue.component ?: ''}\","
                                                    projectCsvContent += "\"${issue.line ?: ''}\","
                                                    projectCsvContent += "\"${(issue.message ?: '').replaceAll('\"','')}\"\n"
                                                }
                                            }
                                        
                                            echo "Added Security Issues To CSV"
                                        }
                                        
                                        /* -------------------------------------------------------- */
                                        
                                        def vulnerabilities = "0"
                                        def bugs = "0"
                                        def codeSmells = "0"
                                        def duplication = "0"
                                        def securityHotspots = "0"
                                        if (json?.component?.measures) {

                                            json.component.measures.each { metric ->
                                        
                                                switch(metric.metric) {
                                        
                                                    case "vulnerabilities":
                                                        vulnerabilities = metric.value ?: "0"
                                                        break
                                        
                                                    case "bugs":
                                                        bugs = metric.value ?: "0"
                                                        break
                                                    case "code_smells":
                                                        codeSmells = metric.value ?: "0"
                                                        break
                                                    
                                                    case "duplicated_lines_density":
                                                        duplication = metric.value ?: "0"
                                                        break
                                        
                                                    case "security_hotspots":
                                                        securityHotspots = metric.value ?: "0"
                                                        break
                                                }
                                            }
                                        }
                                        
                                        /* SEND MAIL HERE */
                                        
                                        def projectCsv = "${project.name}-issues.csv"
                                        
                                        writeFile(
                                            file: projectCsv,
                                            text: projectCsvContent
                                        )
                                        
                                        def projectHtml = """
                                        <html>
                                        <head>
                                        <style>
                                        body {
                                            font-family: Arial, sans-serif;
                                            color: #333333;
                                        }
                                        
                                        table {
                                            border-collapse: collapse;
                                            width: 75%;
                                        }
                                        
                                        th {
                                            background-color: #f2f2f2;
                                        }
                                        
                                        th, td {
                                            border: 1px solid #dddddd;
                                            padding: 10px;
                                            text-align: center;
                                        }
                                        </style>
                                        </head>
                                        
                                        <body>
                                        
                                        <h2>SonarQube Analysis Report - ${project.name}</h2>
                                        
                                        <p>Dear Team,</p>
                                        
                                        <p>
                                        The latest SonarQube analysis for
                                        <b>${project.name}</b>
                                        has been completed successfully.
                                        Please review the quality metrics summarized below.
                                        </p>
                                        
                                        <table>
                                        <tr>
                                            <th>Quality Gate</th>
                                            <th>Vulnerabilities</th>
                                            <th>Bugs</th>
                                            <th>Code Smells</th>
                                            <th>Duplication</th>
                                            <th>Security Hotspots</th>
                                        </tr>
                                        
                                        <tr>
                                            <td><b>${qualityGate}</b></td>
                                            <td>${vulnerabilities}</td>
                                            <td>${bugs}</td>
                                            <td>${codeSmells}</td>
                                            <td>${duplication}%</td>
                                            <td>${securityHotspots}</td>
                                        </tr>
                                        
                                        </table>
                                        
                                        <br>
                                        
                                        <p>
                                        <b>Attached Report:</b><br>
                                        The attached CSV file contains detailed information related to:
                                        </p>
                                        
                                        <ul>
                                            <li>Vulnerabilities</li>
                                            <li>Bugs</li>
                                            <li>Security Hotspots</li>
                                        </ul>
                                        
                                        <p>
                                        <b>Note:</b> Code Smells and Duplication details are not included in the attachment.
                                        Please review the SonarQube dashboard for complete analysis and remediation details.
                                        </p>
                                        
                                        <p>
                                        <b>SonarQube Dashboard:</b><br>
                                        <a href="${SONAR_HOST_URL}/dashboard?id=${project.sonarKey}">
                                        ${project.sonarName}
                                        </a>
                                        </p>
                                        
                                        <br>
                                        
                                        <p>
                                        Kindly review and address the identified issues as part of the ongoing code quality and security improvement process.
                                        </p>
                                        
                                        <br>
                                        
                                        <p>
                                        Regards,<br>
                                        <b>DevOps Team</b>
                                        </p>
                                        
                                        </body>
                                        </html>
                                        """
                                        
                                        emailext(
                                            subject: "${project.name} - SonarQube Analysis Report",
                                            mimeType: 'text/html',
                                            body: projectHtml,
                                            to: "${project.mail},sruthi.d@babujiventures.in",
                                            attachmentsPattern: projectCsv
                                        )
                                        htmlRows += """
                                        <tr>
                                            <td>${project.name}</td>
                                            <td style="color:green;"><b>SUCCESS</b></td>
                                            <td>${qualityGate}</td>
                                            <td>${vulnerabilities}</td>
                                            <td>${bugs}</td>
                                            <td>${securityHotspots}</td>
                                        </tr>
                                        """
                                    }
                                }
                            }

                        } catch (Exception ex) {

                            echo "ERROR IN ${project.name}"
                            echo ex.toString()

                            currentBuild.result = 'UNSTABLE'

                            htmlRows += """
                            <tr>
                                <td>${project.name}</td>
                                <td style="color:red;"><b>FAILED</b></td>
                                <td>N/A</td>
                                <td>N/A</td>
                                <td>N/A</td>
                                <td>N/A</td>
                            </tr>
                            """
                        }
                    }

                    def finalHtml = """
                    <html>
                    <body style="font-family: Arial, sans-serif;">
                    
                    <h2>SonarQube Analysis Summary Report</h2>
                    
                    <p>
                    Dear Team,
                    </p>
                    
                    <p>
                    The scheduled SonarQube analysis has been completed.
                    Please find below the latest quality metrics for the analyzed applications.
                    </p>
                    
                    <br/>
                    
                    <table border="1" cellpadding="8" cellspacing="0">
                    <tr>
                        <th>Project</th>
                        <th>Build Status</th>
                        <th>Quality Gate</th>
                        <th>Vulnerabilities</th>
                        <th>Bugs</th>
                        <th>Security Hotspots</th>
                    </tr>
                    
                    ${htmlRows}
                    
                    </table>
                    
                    <br/>
                    
                    <p>
                    Detailed issue reports have been shared separately with the respective project teams.
                    </p>
                    
                    <p>
                    For complete code quality analysis, including
                    <b>Code Smells</b> and <b>Duplication</b> details,
                    please review the respective SonarQube project dashboards.
                    </p>
                    
                    <br/>
                    
                    <p>Regards,</p>
                    <p><b>DevOps Team</b></p>
                    
                    <img src="https://babujiventures.in/assets/img/clients/baabuji-logo-1-cropped.png"
                         style="height:60px;"/>
                    
                    </body>
                    </html>
                    """
                }
            }
        }
    }
}
