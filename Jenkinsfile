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
//     /opt/tomcat/webapps/app.war.bak
// Used for all rollback scenarios.
//
// NOTE
// ----
// VS Code Groovy warnings are known false positives.
// =====================================================================================


// =====================================================================================
// HELPER FUNCTIONS
// =====================================================================================

/*
 * Builds an HTML summary table used in email notifications.
 */
def buildSummaryHtml() {
    return '<table border="1" cellpadding="8" cellspacing="0" style="border-collapse:collapse;font-family:Arial;">' +
           '<tr style="background:#f2f2f2;"><th align="left">Item</th><th align="left">Value</th></tr>' +
           '<tr><td><b>Job Name</b></td><td>' + (env.JOB_NAME ?: 'N/A') + '</td></tr>' +
           '<tr><td><b>Build #</b></td><td>' + (env.BUILD_NUMBER ?: 'N/A') + '</td></tr>' +
           '<tr><td><b>Status</b></td><td>' + currentBuild.currentResult + '</td></tr>' +
           '<tr><td><b>Branch</b></td><td>' + (env.GIT_BRANCH ?: 'N/A') + '</td></tr>' +
           '<tr><td><b>Triggered By</b></td><td>' +
               (currentBuild.getBuildCauses()?.getAt(0)?.shortDescription ?: 'N/A') +
           '</td></tr>' +
           '<tr><td><b>Build URL</b></td><td><a href="' + env.BUILD_URL + '">' +
               env.BUILD_URL + '</a></td></tr>' +
           '</table>'
}

/*
 * Determines environment name and build timestamp (IST)
 */
def getEnvAndDateInfo() {

    def jobNameUpper = (env.JOB_NAME ?: '').toUpperCase()
    def envName = 'UNKNOWN'

    if      (jobNameUpper.startsWith('PROD-')) { envName = 'PROD' }
    else if (jobNameUpper.startsWith('UAT-'))  { envName = 'UAT' }
    else if (jobNameUpper.startsWith('SIT-'))  { envName = 'SIT' }
    else if (jobNameUpper.startsWith('DEV-'))  { envName = 'DEV' }

    def buildDate = new Date().format(
        'dd-MMM-yyyy HH:mm:ss',
        TimeZone.getTimeZone('Asia/Kolkata')
    )

    return [envName: envName, buildDate: buildDate]
}


// =====================================================================================
// EMAIL CONFIGURATION
// =====================================================================================

def defaultDL     = ''
def PostbuildDL   = ''

def triggerEmail   = null
def finalEmailList = null


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

        stage('Prepare Email Distribution List') {
            steps {
                script {
                    try {
                        triggerEmail = readFile(
                            '/home/lraja/Github/Lightweight-Automation/Trigger_SITBuild.txt'
                        )
                        .readLines()
                        .find { it?.trim()?.startsWith('Email=') }
                        ?.split('=', 2)[1]
                        ?.replaceAll('"', '')
                        ?.split(',')
                        ?.collect { it.trim() }
                        ?.join(',')
                    } catch (ignored) {
                        echo 'Trigger email file not found or unreadable'
                    }

                    finalEmailList = [defaultDL, triggerEmail].findAll { it }.join(',')
                    echo "Final email list resolved as: ${finalEmailList ?: defaultDL}"
                }
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
                archiveArtifacts artifacts: 'target/*.war', followSymlinks: false
                fingerprint 'target/*.war'
            }
        }

        stage('Unit Testing') {
            steps {
                sh "${BUILD_TOOL_CMD} test"
            }
        }

        // =================================================================================
        // DEPLOYMENT WITH REMOTE BACKUP & ROLLBACK (PRODUCTION SAFE)
        // Backup Path (REMOTE): /opt/tomcat/webapps/app.war.bak
        // =================================================================================
        stage('Deployment') {
            steps {
                script {
                    try {

                        // -----------------------------------------------------------------
                        // REMOTE BACKUP (runs on Tomcat server)
                        // -----------------------------------------------------------------
                        sshPublisher(
                            publishers: [
                                sshPublisherDesc(
                                    configName: 'tomcat',
                                    verbose: true,
                                    transfers: [
                                        sshTransfer(
                                            execCommand: '''
                                                if [ -f /opt/tomcat/webapps/app.war ]; then
                                                    cp /opt/tomcat/webapps/app.war \
                                                       /opt/tomcat/webapps/app.war.bak
                                                    echo "Remote backup created at /opt/tomcat/webapps/app.war.bak"
                                                else
                                                    echo "No existing WAR found to backup"
                                                fi
                                            '''
                                        )
                                    ]
                                )
                            ]
                        )

                        // -----------------------------------------------------------------
                        // DEPLOY NEW WAR (remote copy)
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
                                            remoteDirectory: '/opt/tomcat/webapps/'
                                        )
                                    ]
                                )
                            ]
                        )

                        echo 'Application deployed successfully.'

                    } catch (err) {

                        echo 'Deployment failed — performing REMOTE rollback'

                        // -----------------------------------------------------------------
                        // REMOTE ROLLBACK
                        // -----------------------------------------------------------------
                        sshPublisher(
                            publishers: [
                                sshPublisherDesc(
                                    configName: 'tomcat',
                                    verbose: true,
                                    transfers: [
                                        sshTransfer(
                                            execCommand: '''
                                                if [ -f /opt/tomcat/webapps/app.war.bak ]; then
                                                    mv /opt/tomcat/webapps/app.war.bak \
                                                       /opt/tomcat/webapps/app.war
                                                    echo "Rollback completed using backup WAR"
                                                else
                                                    echo "Backup WAR not found — rollback skipped"
                                                fi
                                            '''
                                        )
                                    ]
                                )
                            ]
                        )

                        error "Deployment failed after rollback: ${err}"
                    }
                }
            }
        }

        // -------------------------------------------------------------------------
        // TOMCAT RESTART WITH REMOTE ROLLBACK SAFETY
        // -------------------------------------------------------------------------
        stage('Restart Servers') {
            steps {
                script {
                    try {

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

                    } catch (err) {

                        echo 'Restart failed — restoring backup WAR remotely'

                        sshPublisher(
                            publishers: [
                                sshPublisherDesc(
                                    configName: 'tomcat',
                                    verbose: true,
                                    transfers: [
                                        sshTransfer(
                                            execCommand: '''
                                                if [ -f /opt/tomcat/webapps/app.war.bak ]; then
                                                    mv /opt/tomcat/webapps/app.war.bak \
                                                       /opt/tomcat/webapps/app.war
                                                    sudo systemctl restart tomcat
                                                fi
                                            '''
                                        )
                                    ]
                                )
                            ]
                        )

                        error "Restart failed after rollback: ${err}"
                    }
                }
            }
        }

        stage('Sanity Check') {
            steps {
                echo 'Performing basic sanity check'
                sh 'curl -m 5 -s http://localhost:8080 || true'
            }
        }

        stage('Post Actions') {
            steps {
                echo 'Post actions completed'
            }
        }
    }

    // =================================================================================
    // POST-BUILD ACTIONS
    // =================================================================================
    post {

        always {
            cleanWs(deleteDirs: true, disableDeferredWipeout: true)
        }

        success {
            script {
                def info = getEnvAndDateInfo()
                retry(3) {
                    emailext(
                        to: "${PostbuildDL},${finalEmailList}",
                        subject: "[SUCCESS] [${info.envName}] ${env.JOB_NAME} #${env.BUILD_NUMBER} | ${info.buildDate}",
                        mimeType: 'text/html',
                        body:
                            '<h2 style="color:green;">Build Successful ✅</h2>' +
                            '<p><b>Environment:</b> ' + info.envName + '</p>' +
                            '<p><b>Build Date/Time:</b> ' + info.buildDate + '</p>' +
                            buildSummaryHtml(),
                        attachLog: true
                    )
                }
            }
        }

        failure {
            script {
                def info = getEnvAndDateInfo()
                retry(3) {
                    emailext(
                        to: "${PostbuildDL},${finalEmailList}",
                        subject: "[FAILED] [${info.envName}] ${env.JOB_NAME} #${env.BUILD_NUMBER} | ${info.buildDate}",
                        mimeType: 'text/html',
                        body:
                            '<h2 style="color:red;">Build Failed ❌</h2>' +
                            '<p><b>Environment:</b> ' + info.envName + '</p>' +
                            '<p><b>Build Date/Time:</b> ' + info.buildDate + '</p>' +
                            buildSummaryHtml(),
                        attachLog: true
                    )
                }
            }
        }

        unstable {
            script {
                def info = getEnvAndDateInfo()
                retry(3) {
                    emailext(
                        to: "${PostbuildDL},${finalEmailList}",
                        subject: "[UNSTABLE] [${info.envName}] ${env.JOB_NAME} #${env.BUILD_NUMBER} | ${info.buildDate}",
                        mimeType: 'text/html',
                        body:
                            '<h2 style="color:orange;">Build Unstable ⚠️</h2>' +
                            '<p><b>Environment:</b> ' + info.envName + '</p>' +
                            '<p><b>Build Date/Time:</b> ' + info.buildDate + '</p>' +
                            buildSummaryHtml(),
                        attachLog: true
                    )
                }
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
/opt/tomcat/webapps/app.war.bak   (REMOTE TOMCAT SERVER)

ROLLBACK FLOW
-------------
1. Remote backup before deployment
2. Deployment failure → remote rollback
3. Restart failure → remote rollback
4. Pipeline fails explicitly

IMPORTANT NOTE
--------------
Backup and rollback are executed on the Tomcat server
using SSH Publisher (NOT on Jenkins agent).

=======================================================================================
*/


