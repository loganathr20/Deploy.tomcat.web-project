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
// Existing WAR is backed up to:
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
 * Keeps email content consistent across all build results.
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
 * Determines environment name from job naming convention
 * and formats build date/time in IST timezone.
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

// Static/default email distribution list
def defaultDL = ''

// Post-build distribution list (Ops / Release / Support)
def PostbuildDL = ''

// Dynamically resolved email lists
def triggerEmail   = null
def finalEmailList = null


// =====================================================================================
// PIPELINE DEFINITION
// =====================================================================================

pipeline {

    // Run on any available Jenkins agent
    agent any

    // Global tools and environment variables
    environment {
        JAVA_HOME      = tool 'JDK'
        BUILD_TOOL_CMD = 'mvn'
    }

    stages {

        // -------------------------------------------------------------------------
        // Displays branch information for debugging and audit clarity
        // -------------------------------------------------------------------------
        stage('Debug Branch Info') {
            steps {
                echo "Running Jenkinsfile from branch: ${env.GIT_BRANCH}"
            }
        }

        // -------------------------------------------------------------------------
        // Resolves email recipients from external trigger file
        // Falls back gracefully if file is missing
        // -------------------------------------------------------------------------
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

        // -------------------------------------------------------------------------
        // Source code checkout from configured SCM
        // -------------------------------------------------------------------------
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        // -------------------------------------------------------------------------
        // Maven build and artifact archival
        // WAR is fingerprinted for traceability
        // -------------------------------------------------------------------------
        stage('Build') {
            steps {
                sh "${BUILD_TOOL_CMD} clean install -DskipTests"
                archiveArtifacts artifacts: 'target/*.war', followSymlinks: false
                fingerprint 'target/*.war'
            }
        }

        // -------------------------------------------------------------------------
        // Executes unit tests
        // -------------------------------------------------------------------------
        stage('Unit Testing') {
            steps {
                sh "${BUILD_TOOL_CMD} test"
            }
        }

        // -------------------------------------------------------------------------
        // Deployment with backup and rollback safety
        // Backup Path: /opt/tomcat/webapps/app.war.bak
        // -------------------------------------------------------------------------
        stage('Deployment') {
            steps {
                script {
                    try {

                        // Backup existing WAR before deploying new artifact
                        sh '''
                            if [ -f /opt/tomcat/webapps/app.war ]; then
                                cp /opt/tomcat/webapps/app.war \
                                   /opt/tomcat/webapps/app.war.bak
                                echo "Backup created at /opt/tomcat/webapps/app.war.bak"
                            fi
                        '''

                        // Deploy WAR to Tomcat using SSH Publisher
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

                        // Restore backup WAR if deployment fails
                        echo 'Deployment failed — performing rollback'

                        sh '''
                            if [ -f /opt/tomcat/webapps/app.war.bak ]; then
                                mv /opt/tomcat/webapps/app.war.bak \
                                   /opt/tomcat/webapps/app.war
                                echo "Rollback completed"
                            fi
                        '''

                        error "Deployment failed after rollback: ${err}"
                    }
                }
            }
        }

        // -------------------------------------------------------------------------
        // Restarts Tomcat and rolls back if restart fails
        // -------------------------------------------------------------------------
        stage('Restart Servers') {
            steps {
                script {
                    try {
                        sh 'sudo systemctl restart tomcat'
                        sh 'sudo systemctl status tomcat'
                    } catch (err) {

                        echo 'Restart failed — restoring backup WAR'

                        sh '''
                            if [ -f /opt/tomcat/webapps/app.war.bak ]; then
                                mv /opt/tomcat/webapps/app.war.bak \
                                   /opt/tomcat/webapps/app.war
                                sudo systemctl restart tomcat
                            fi
                        '''

                        error "Restart failed after rollback: ${err}"
                    }
                }
            }
        }

        // -------------------------------------------------------------------------
        // Lightweight sanity validation (non-blocking)
        // -------------------------------------------------------------------------
        stage('Sanity Check') {
            steps {
                script {
                    echo 'Performing basic sanity check'
                    sh 'curl -m 5 -s http://localhost:8080 || true'
                }
            }
        }

        // -------------------------------------------------------------------------
        // Logical end-of-pipeline marker
        // -------------------------------------------------------------------------
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

        // Workspace cleanup always runs
        always {
            cleanWs(deleteDirs: true, disableDeferredWipeout: true)
        }

        // Success notification
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

        // Failure notification
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

        // Unstable notification
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
OVERVIEW
--------
This Jenkins pipeline builds, tests, deploys, and safely manages a Java WAR
deployment to Tomcat with automatic rollback and HTML notifications.

BACKUP PATH
-----------
/opt/tomcat/webapps/app.war.bak

ROLLBACK FLOW
-------------
1. Backup created before deployment
2. Deployment failure → backup restored
3. Restart failure → backup restored
4. Pipeline fails explicitly

EMAIL BEHAVIOR
--------------
- Post-build only
- HTML formatted
- Retry-safe (3 attempts)
- Console log attached

PIPELINE FLOW
-------------
Debug → Email DL → Checkout → Build → Test
→ Deploy → Restart → Sanity → Post → Email → Cleanup

NOTES
-----
- Safe for DEV / SIT / UAT / PROD
- README comments do not affect execution

=======================================================================================
*/

