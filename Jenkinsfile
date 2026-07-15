import groovy.json.JsonSlurperClassic

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
                            sonarName: 'CMS-Dev_Backend'
                        ]
                    ]

                    def htmlRows = ""
                    def csvContent = "Project,Severity,Type,File,Line,Message,Status\n"
                    projects.each { project ->

                        echo "================================================="
                        echo "Processing ${project.name}"
                        echo "================================================="

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
                                        
                                        def json = new JsonSlurperClassic().parseText(response)
                                        
                                        /* ------------------- GET SONAR ISSUES ------------------- */
                                        
                                        def issuesResponse = sh(
                                            script: """
                                                curl -s -u \$SONAR_TOKEN: \
                                                "${SONAR_HOST_URL}/api/issues/search?componentKeys=${project.sonarKey}&resolved=false&ps=500"
                                            """,
                                            returnStdout: true
                                        ).trim()
                                        
                                        echo "Issues Response Length: ${issuesResponse.length()}"
                                        
                                        def issuesJson = new JsonSlurperClassic().parseText(issuesResponse)
                                        
                                        if (issuesJson?.issues) {
                                        
                                            issuesJson.issues.each { issue ->
                                        
                                                csvContent += "\"${project.name}\","
                                                csvContent += "\"${issue.severity ?: ''}\","
                                                csvContent += "\"${issue.type ?: ''}\","
                                                csvContent += "\"${issue.component ?: ''}\","
                                                csvContent += "\"${issue.line ?: ''}\","
                                                csvContent += "\"${(issue.message ?: '').replaceAll('\"','')}\","
                                                csvContent += "\"${issue.status ?: ''}\"\n"
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
                                                        securityHotspots = metric.value
                                                        break
                                                }
                                            }
                                        }

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

                    writeFile(
                        file: 'sonar-report.html',
                        text: finalHtml
                    )
                    writeFile(
                        file: 'sonar-issues.csv',
                        text: csvContent
                    )
                }
            }
        }
    }

    post {
        always {
            script {

                def mailBody = readFile('sonar-report.html')

                emailext(
                    subject: "Multi Project SonarQube Report | ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    mimeType: 'text/html',
                    body: mailBody,
                    to: 'sruthi.d@babujiventures.in',
                    attachmentsPattern: 'sonar-issues.csv'
                )
            }
        }
    }
}
