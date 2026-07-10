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
                        name: 'ITCMS',
                        repo: 'https://github.com/Baabujiventuress/ITCMS_Springboot_Backend',
                        branch: 'ITCMS-DEV',
                        credentials: 'rajeshkanna',
                        sonarKey: 'ITCM-Dev_Backend',
                        sonarName: 'ITCM-Dev_Backend'
                    ],

                    [
                        name: 'CMS',
                        repo: 'https://github.com/Baabujiventuress/CMS_Springboot_Backend',
                        branch: 'v1-dev-branch',
                        credentials: 'rajeshkanna',
                        sonarKey: 'CMS-Dev_Backend',
                        sonarName: 'CMS-Dev_Backend'
                    ],

                    [
                        name: 'Bustle',
                        repo: 'https://github.com/Baabujiventuress/Bustle_Springboot_Backend',
                        branch: 'bustle-dev',
                        credentials: 'raghul',
                        sonarKey: 'Bustle-Dev_Backend',
                        sonarName: 'Bustle-Dev_Backend'
                    ],

                    [
                        name: 'QuickRentPay',
                        repo: 'https://github.com/Baabujiventuress/QuickRentPay_Springboot_Backend',
                        branch: 'staging_build',
                        credentials: 'prakash',
                        sonarKey: 'Quickrentpay-Dev_Backend',
                        sonarName: 'Quickrentpay-Dev_Backend'
                    ],

                    [
                        name: 'SCV',
                        repo: 'https://github.com/Baabujiventuress/SCV_Partner_Onboarding_Backend.git',
                        branch: 'Staging',
                        credentials: 'prakash',
                        sonarKey: 'SCV-Dev_Backend',
                        sonarName: 'SCV-Dev_Backend'
                    ],

                    [
                        name: 'Facheck',
                        repo: 'https://github.com/Baabujiventuress/Facheck_Springboot_Backend',
                        branch: 'Facheck-Dev-DailyBuild',
                        credentials: 'abhishek',
                        sonarKey: 'Facheck-Dev_Backend',
                        sonarName: 'Facheck-Dev_Backend'
                    ]
                ]
                def htmlRows = ""
                def gradeMap = [
                    '1':'A',
                    '1.0':'A',
                    '2':'B',
                    '2.0':'B',
                    '3':'C',
                    '3.0':'C',
                    '4':'D',
                    '4.0':'D',
                    '5':'E',
                    '5.0':'E'
                ]

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
                                def qg = waitForQualityGate(
                                    abortPipeline: false
                                )
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
                                            "${SONAR_HOST_URL}/api/measures/component?component=${project.sonarKey}&metricKeys=vulnerabilities,bugs,code_smells,coverage,duplicated_lines_density,security_hotspots_reviewed"
                                        """,
                                        returnStdout: true
                                    ).trim()

                                    def vulnerabilities = "0"
                                    def bugs = "0"
                                    def codeSmells = "0"
                                    def coverage = "0"
                                    def duplication = "0"
                                    def hotspots = "0"

                                    json.component.measures.each { metric ->
                                        switch(metric.metric) {

    case "vulnerabilities":
        vulnerabilities = metric.value
        break

    case "bugs":
        bugs = metric.value
        break

    case "code_smells":
        codeSmells = metric.value
        break

    case "coverage":
        coverage = metric.value
        break

    case "duplicated_lines_density":
        duplication = metric.value
        break

    case "security_hotspots_reviewed":
        hotspots = metric.value
        break
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
    <td>${hotspots}%</td>
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
    <th>Hotspots Reviewed</th>
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
                to: 'sruthi.d@babujiventures.in'
            )
        }
    }
}

}
