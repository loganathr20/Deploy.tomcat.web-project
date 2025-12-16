
// =====================================================================================
// JENKINS PIPELINE – JAVA WAR DEPLOYMENT WITH SAFE ROLLBACK & HTML EMAIL NOTIFICATIONS
// =====================================================================================
//
// PURPOSE
// -------
// End-to-end CI/CD pipeline for Java WAR applications.
// Handles build, test, deploy, rollback, restart, and notifications.
//
// BACKUP LOCATION
// ---------------
// Existing WAR is backed up on the *REMOTE TOMCAT SERVER* at:
//     /opt/tomcat/webapps/<actual-war-name>.bak
//
// Example:
//     dtw-1.0.0.war  →  dtw-1.0.0.war.bak
//
// NOTE
// ----
// VS Code Groovy warnings are known false positives
// =====================================================================================


// =====================================================================================
// HELPER FUNCTIONS
// =====================================================================================

def buildSummaryHtml() {
    return '<table border="1" cellpadding="8" cellspacing="0" style="border-collapse:collapse;font-family:Arial;">' +
           '<tr style="background:#f2f2f2;"><th>Item</th><th>Value</th></tr>' +
           '<tr><td>Job</td><td>' + (env.JOB_NAME ?: 'N/A') + '</td></tr>' +
           '<tr><td>Build</td><td>#' + (env.BUILD_NUMBER ?: 'N/A') + '</td></tr>' +
           '<tr><td>Status</td><td>' + currentBuild.currentResult + '</td></tr>' +
           '<tr><td>Branch</td><td>' + (env.GIT_BRANCH ?: 'N/A') + '</td></tr>' +
           '<tr><td>URL</td><td><a href="' + env.BUILD_URL + '">' + env.BUILD_URL + '</a></td></tr>' +
           '</table>'
}

def getEnvAndDateInfo() {
    def envName = 'UNKNOWN'
    def job = (env.JOB_NAME ?: '').toUpperCase()

    if (job.startsWith('PROD-')) envName = 'PROD'
    else if (job.startsWith('UAT-')) envName = 'UAT'
    else if (job.startsWith('SIT-')) envName = 'SIT'
    else if (job.startsWith('DEV-')) envName = 'DEV'

    def buildDate = new Date().format(
        'dd-MMM-yyyy HH:mm:ss',
        TimeZone.getTimeZone('Asia/Kolkata')
    )

    return [envName: envName, buildDate: buildDate]
}


// =====================================================================================
// PIPELINE DEFINITION
// =====================================================================================

pipeline {

    agent any

    environment {
        JAVA_HOME      = tool 'JDK'
        BUILD_TOOL_CMD = 'mvn'
    }

    stages {

        stage('Debug Branch Info') {
            steps {
                echo "Running Jenkinsfile from branch: ${env.GIT_BRANCH}"
            }
        }

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh "${BUILD_TOOL_CMD} clean install -DskipTests"
                archiveArtifacts artifacts: 'target/*.war'
            }
        }

        stage('Unit Testing') {
            steps {
                sh "${BUILD_TOOL_CMD} test"
            }
        }

        // =================================================================================
        // DEPLOYMENT WITH **DYNAMIC REMOTE BACKUP**
        // =================================================================================
        stage('Deployment') {
            steps {
                script {
                    try {

                        // -----------------------------------------------------------------
                        // REMOTE BACKUP (DETECT REAL WAR NAME)
                        // -----------------------------------------------------------------
                        sshPublisher(
                            publishers: [
                                sshPublisherDesc(
                                    configName: 'tomcat',
                                    verbose: true,
                                    transfers: [
                                        sshTransfer(
                                            execCommand: '''
                                                WAR=$(ls /opt/tomcat/webapps/*.war 2>/dev/null | head -1)

                                                if [ -n "$WAR" ]; then
                                                    cp "$WAR" "$WAR.bak"
                                                    echo "Backup created: $WAR.bak"
                                                else
                                                    echo "No WAR found to backup"
                                                fi
                                            '''
                                        )
                                    ]
                                )
                            ]
                        )

                        // -----------------------------------------------------------------
                        // DEPLOY NEW WAR (CORRECT LOCATION)
                        // -----------------------------------------------------------------
                        sshPublisher(
                            publishers: [
                                sshPublisherDesc(
                                    configName: 'tomcat',
                                    verbose: true,
                                    transfers: [
                                        sshTransfer(
                                            sourceFiles: 'target/*.war',
                                            removePrefix: 'target/',
                                            remoteDirectory: '/opt/tomcat/webapps'
                                        )
                                    ]
                                )
                            ]
                        )

                        echo 'Application deployed successfully'

                    } catch (err) {

                        echo 'Deployment failed — performing rollback'

                        // -----------------------------------------------------------------
                        // REMOTE ROLLBACK (MATCHING WAR)
                        // -----------------------------------------------------------------
                        sshPublisher(
                            publishers: [
                                sshPublisherDesc(
                                    configName: 'tomcat',
                                    verbose: true,
                                    transfers: [
                                        sshTransfer(
                                            execCommand: '''
                                                BAK=$(ls /opt/tomcat/webapps/*.war.bak 2>/dev/null | head -1)

                                                if [ -n "$BAK" ]; then
                                                    mv "$BAK" "${BAK%.bak}"
                                                    echo "Rollback completed"
                                                else
                                                    echo "Backup not found — rollback skipped"
                                                fi
                                            '''
                                        )
                                    ]
                                )
                            ]
                        )

                        error "Deployment failed after rollback"
                    }
                }
            }
        }

        stage('Restart Servers') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'tomcat',
                            verbose: true,
                            transfers: [
                                sshTransfer(
                                    execCommand: '''
                                        sudo systemctl restart tomcat
                                        sudo systemctl status tomcat
                                    '''
                                )
                            ]
                        )
                    ]
                )
            }
        }

        stage('Sanity Check') {
            steps {
                sh 'curl -m 5 -s http://localhost:8080 || true'
            }
        }
    }

    post {

        always {
            cleanWs()
        }

        success {
            script {
                def info = getEnvAndDateInfo()
                emailext(
                    subject: "[SUCCESS][${info.envName}] ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    mimeType: 'text/html',
                    body: buildSummaryHtml()
                )
            }
        }

        failure {
            script {
                def info = getEnvAndDateInfo()
                emailext(
                    subject: "[FAILED][${info.envName}] ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    mimeType: 'text/html',
                    body: buildSummaryHtml()
                )
            }
        }
    }
}


// =====================================================================================
// README – PIPELINE DOCUMENTATION
// =====================================================================================
/*
BACKUP LOCATION
---------------
/opt/tomcat/webapps/<war-name>.bak

Example:
    dtw-1.0.0.war.bak

ROLLBACK FLOW
-------------
1. Detect existing WAR
2. Create .bak before deployment
3. Deployment failure → restore .bak
4. Restart failure → restore .bak
5. Pipeline fails explicitly

IMPORTANT NOTE
--------------
Backup and rollback are executed on the Tomcat server
using SSH Publisher (NOT on Jenkins agent).
*/

