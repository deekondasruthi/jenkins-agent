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
                            name: 'CMS',
                            repo: 'https://github.com/Baabujiventuress/CMS_Springboot_Backend',
                            branch: 'v1-dev-branch',
                            credentials: 'rajeshkanna',
                            sonarKey: 'CMS-Dev_Backend',
                            sonarName: 'CMS-Dev_Backend',
                            mail: 'rajeshkanna.m@babujiventures.in'
                        ],
                        [
                            name: 'QuickRentPay',
                            repo: 'https://github.com/Baabujiventuress/QuickRentPay_Springboot_Backend',
                            branch: 'staging_build',
                            credentials: 'prakash',
                            sonarKey: 'Quickrentpay-Dev_Backend',
                            sonarName: 'Quickrentpay-Dev_Backend',
                            mail: 'prakash.p@babujiventures.in'
                        ],
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
                            "Project,Severity,Type,File,Line,Message,Status\n"
                        
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
                                                "${SONAR_HOST_URL}/api/measures/component?component=${project.sonarKey}&metricKeys=vulnerabilities,bugs,code_smells,coverage,duplicated_lines_density,security_hotspots"
                                            """,
                                            returnStdout: true
                                        ).trim()
                                        
                                        echo "Sonar Response: ${response}"
                                        
                                        def json = readJSON text: response
                                        
                                        /* ------------------- GET SONAR ISSUES ------------------- */
                                        
                                        def issuesResponse = sh(
                                            script: """
                                                curl -s -u \$SONAR_TOKEN: \
                                                "${SONAR_HOST_URL}/api/issues/search?componentKeys=${project.sonarKey}&resolved=false&ps=10000"
                                            """,
                                            returnStdout: true
                                        ).trim()
                                        
                                        echo "Issues Response Length: ${issuesResponse.length()}"
                                        
                                        def issuesJson = readJSON text: issuesResponse
                                        
                                        if (issuesJson?.issues) {
                                        
                                            issuesJson.issues.each { issue ->
                                        
                                                projectCsvContent += "\"${project.name}\","
                                                projectCsvContent += "\"${issue.severity ?: ''}\","
                                                projectCsvContent += "\"${issue.type ?: ''}\","
                                                projectCsvContent += "\"${issue.component ?: ''}\","
                                                projectCsvContent += "\"${issue.line ?: ''}\","
                                                projectCsvContent += "\"${(issue.message ?: '').replaceAll('\"','')}\","
                                                projectCsvContent += "\"${issue.status ?: ''}\"\n"
                                            }
                                        
                                            echo "Added ${issuesJson.issues.size()} issues to CSV"
                                        }
                                        
                                        /* -------------------------------------------------------- */
                                        
                                        def vulnerabilities = "0"
                                        def bugs = "0"
                                        def codeSmells = "0"
                                        def coverage = "0"
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
                                        
                                                    case "coverage":
                                                        coverage = metric.value ?: "0"
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
                                        <body>
                                        
                                        <h2>${project.name} SonarQube Report</h2>
                                        
                                        <p>
                                        Build Number :
                                        <b>${env.BUILD_NUMBER}</b>
                                        </p>
                                        
                                        <p>
                                        Quality Gate :
                                        <b>${qualityGate}</b>
                                        </p>
                                        
                                        <br/>
                                        
                                        <table border="1" cellpadding="8" cellspacing="0">
                                        <tr>
                                            <th>Vulnerabilities</th>
                                            <th>Bugs</th>
                                            <th>Code Smells</th>
                                            <th>Coverage</th>
                                            <th>Duplication</th>
                                            <th>Security Hotspots</th>
                                        </tr>
                                        
                                        <tr>
                                            <td>${vulnerabilities}</td>
                                            <td>${bugs}</td>
                                            <td>${codeSmells}</td>
                                            <td>${coverage}%</td>
                                            <td>${duplication}%</td>
                                            <td>${securityHotspots}</td>
                                        </tr>
                                        
                                        </table>
                                        
                                        <br/>
                                        
                                        <p>
                                        Dashboard:
                                        <a href="${SONAR_HOST_URL}/dashboard?id=${project.sonarKey}">
                                        ${project.sonarName}
                                        </a>
                                        </p>
                                        
                                        <br/>
                                        
                                        <p>
                                        Attached CSV contains Sonar issues for ${project.name}.
                                        </p>
                                        
                                        </body>
                                        </html>
                                        """
                                        
                                        emailext(
                                            subject: "${project.name} SonarQube Report | Build #${env.BUILD_NUMBER}",
                                            mimeType: 'text/html',
                                            body: projectHtml,
                                            to: "${project.mail},sruthi.d@babujiventures.in,Faisal.Ahmed@basispay.in",
                                            attachmentsPattern: projectCsv
                                        )
                                        htmlRows += """
                                        <tr>
                                            <td>${project.name}</td>
                                            <td style="color:green;"><b>SUCCESS</b></td>
                                            <td>${qualityGate}</td>
                                            <td>${vulnerabilities}</td>
                                            <td>${bugs}</td>
                                            <td>${codeSmells}</td>
                                            <td>${coverage}%</td>
                                            <td>${duplication}%</td>
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
    <td>N/A</td>
    <td>N/A</td>
    <td>N/A</td>
</tr>
"""
                        }
                    }

                    def finalHtml = """
<html>
<body>

<h2>Multi Project SonarQube Report</h2>

<p>
Build Number :
<b>${env.BUILD_NUMBER}</b>
</p>

<p>
Job Name :
<b>${env.JOB_NAME}</b>
</p>

<p>
Result :
<b>${currentBuild.currentResult}</b>
</p>

<br/>

<table border="1" cellpadding="8" cellspacing="0">
<tr>
    <th>Project</th>
    <th>Build Status</th>
    <th>Quality Gate</th>
    <th>Vulnerabilities</th>
    <th>Bugs</th>
    <th>Code Smells</th>
    <th>Coverage</th>
    <th>Duplication</th>
    <th>Security Hotspots</th>
</tr>

${htmlRows}

</table>

<br/><br/>

<p>Regards,</p>
<p>DevOps Team</p>

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
