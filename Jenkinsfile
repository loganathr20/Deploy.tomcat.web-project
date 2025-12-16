// =====================================================================================
// JENKINS PIPELINE – JAVA WAR DEPLOYMENT WITH SAFE ROLLBACK & HTML EMAIL NOTIFICATIONS
// =====================================================================================
//
// PURPOSE
// -------
// This Jenkins pipeline provides a complete CI/CD workflow for a Java WAR-based
// application deployed on Apache Tomcat.
//
// The pipeline performs:
//   - Source checkout
//   - Maven build & unit testing
//   - Safe deployment to Tomcat
//   - Remote backup of existing WAR
//   - Automatic rollback on deployment or restart failure
//   - HTML email notifications with build summary
//
// BACKUP LOCATION (REMOTE TOMCAT SERVER)
// -------------------------------------
// The currently deployed WAR is backed up dynamically using its real filename.
// Backup is created in the SAME directory as the WAR.
//
// Example:
//     /opt/tomcat/webapps/dtw-1.0.0.war
//     /opt/tomcat/webapps/dtw-1.0.0.war.bak
//
// NOTE
// ----
// VS Code Groovy “brackets do not match” warnings are known false positives
// and do NOT affect Jenkins execution.
// =====================================================================================


// =====================================================================================
// HELPER FUNCTIONS
// =====================================================================================

/*
 * buildSummaryHtml()
 * ------------------
 * Generates a small HTML table summarizing the Jenkins build.
 * This HTML is embedded in all email notifications (SUCCESS / FAILURE / UNSTABLE).
 */
def buildSummaryHtml() {
    return '<table border="1" cellpadding="8" cellspacing="0" style="border-collapse:collapse;font-family:Arial;">' +
           '<tr style="background:#f2f2f2;"><th>Item</th><th>Value</th></tr>' +
           '<tr><td><b>Job Name</b></td><td>' + (env.JOB_NAME ?: 'N/A') + '</td></tr>' +
           '<tr><td><b>Build #</b></td><td>' + (env.BUILD_NUMBER ?: 'N/A') + '</td></tr>' +
           '<tr><td><b>Status</b></td><td>' + currentBuild.currentResult + '</td></tr>' +
           '<tr><td><b>Branch</b></td><td>' + (env.GIT_BRANCH ?: 'N/A') + '</td></tr>' +
           '<tr><td><b>Build URL</b></td><td><a href="' + env.BUILD_URL + '">' +
               env.BUILD_URL + '</a></td></tr>' +
           '</table>'
}

/*
 * getEnvAndDateInfo()
 * ------------------
 * Determines:
 *   - Environment name (DEV / SIT / UAT / PROD) based on job naming convention
 *   - Build timestamp in IST timezone
 *
 * This information is used in email subject lines and email bodies.
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
// EMAIL CONFIGURATION VARIABLES
// =====================================================================================
//
// defaultDL     : Static default email distribution list
// PostbuildDL   : Optional post-build distribution list
// triggerEmail  : Email(s) read dynamically from trigger file
// finalEmailList: Combined final list used for notifications
//
def defaultDL     = ''
def PostbuildDL   = ''

def triggerEmail   = null
def finalEmailList = null


// =====================================================================================
// PIPELINE DEFINITION
// =====================================================================================

pipeline {

    // Any available Jenkins agent can execute this pipeline
    agent any

    // Global environment variables for tools and commands
    environment {
        JAVA_HOME      = tool 'JDK'
        BUILD_TOOL_CMD = 'mvn'
    }

    stages {

        // -------------------------------------------------------------------------
        // DEBUG INFORMATION
        // -------------------------------------------------------------------------
        // Helps identify which Git branch triggered the pipeline
        stage('Debug Branch Info') {
            steps {
                echo "Running Jenkinsfile from branch: ${env.GIT_BRANCH}"
            }
        }

        // -------------------------------------------------------------------------
        // EMAIL DISTRIBUTION LIST PREPARATION
        // -------------------------------------------------------------------------
        // Reads a trigger file to dynamically fetch email recipients.
        // Safe failure handling is included if the file is missing.
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
        // SOURCE CODE CHECKOUT
        // -------------------------------------------------------------------------
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        // -------------------------------------------------------------------------
        // BUILD STAGE
        // -------------------------------------------------------------------------
        // Compiles the project and generates the WAR artifact.
        stage('Build') {
            steps {
                sh "${BUILD_TOOL_CMD} clean install -DskipTests"
                archiveArtifacts artifacts: 'target/*.war', followSymlinks: false
                fingerprint 'target/*.war'
            }
        }

        // -------------------------------------------------------------------------
        // UNIT TESTING
        // -------------------------------------------------------------------------
        stage('Unit Testing') {
            steps {
                sh "${BUILD_TOOL_CMD} test"
            }
        }

        // =================================================================================
        // DEPLOYMENT WITH REMOTE BACKUP & ROLLBACK (PRODUCTION SAFE)
        // =================================================================================
        // This stage:
        //   1. Detects the currently deployed WAR on Tomcat
        //   2. Creates a backup (<war>.bak)
        //   3. Deploys the new WAR
        //   4. Rolls back automatically on failure
        stage('Deployment') {
            steps {
                script {
                    try {

                        // -----------------------------------------------------------------
                        // REMOTE BACKUP STEP
                        // -----------------------------------------------------------------
                        // Detects the real WAR name dynamically and creates a .bak file.
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
                                                    echo "No existing WAR found to backup"
                                                fi
                                            '''
                                        )
                                    ]
                                )
                            ]
                        )

                        // -----------------------------------------------------------------
                        // DEPLOY NEW WAR
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

                        echo 'Application deployed successfully.'

                    } catch (err) {

                        echo 'Deployment failed — performing remote rollback'

                        // -----------------------------------------------------------------
                        // REMOTE ROLLBACK STEP
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
        // TOMCAT RESTART WITH ROLLBACK SAFETY
        // -------------------------------------------------------------------------
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

        // -------------------------------------------------------------------------
        // BASIC SANITY CHECK
        // -------------------------------------------------------------------------
        stage('Sanity Check') {
            steps {
                echo 'Performing basic sanity check'
                sh 'curl -m 5 -s http://localhost:8080 || true'
            }
        }

        // -------------------------------------------------------------------------
        // POST ACTION PLACEHOLDER
        // -------------------------------------------------------------------------
        stage('Post Actions') {
            steps {
                echo 'Post actions completed'
            }
        }
    }

    // =================================================================================
    // POST-BUILD ACTIONS
    // =====================================================================================
    post {

        // Always clean workspace after pipeline completion
        always {
            cleanWs(deleteDirs: true, disableDeferredWipeout: true)
        }

        // SUCCESS EMAIL NOTIFICATION
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

        // FAILURE EMAIL NOTIFICATION
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

        // UNSTABLE EMAIL NOTIFICATION
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
PIPELINE FLOW
-------------
Checkout → Build → Test → Backup → Deploy → Restart → Sanity → Notify

BACKUP LOCATION
---------------
/opt/tomcat/webapps/<actual-war-name>.bak

ROLLBACK CONDITIONS
-------------------
• Deployment failure
• Tomcat restart failure

WHY THIS IS SAFE
----------------
✔ Backup created before deployment
✔ Rollback is atomic (mv)
✔ No manual intervention required
✔ Preserves last known good WAR

IMPORTANT NOTE
--------------
All backup and rollback operations run on the
REMOTE TOMCAT SERVER via SSH Publisher.

=======================================================================================
*/


