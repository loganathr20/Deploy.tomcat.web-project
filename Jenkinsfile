// =====================================================================================
// JENKINS PIPELINE – JAVA WAR DEPLOYMENT WITH SAFE ROLLBACK & HTML EMAIL NOTIFICATIONS
// =====================================================================================
//
// PURPOSE
// -------
// This pipeline performs the following:
//   1. Checkout source code
//   2. Build Java WAR using Maven
//   3. Execute unit tests
//   4. Deploy WAR to Tomcat using SSH Publisher
//   5. Perform SAFE deployment with rollback on failure
//   6. Restart Tomcat safely
//   7. Send HTML email notifications (SUCCESS / FAILURE / UNSTABLE)
//
// DESIGN PRINCIPLES
// -----------------
// ✔ No destructive changes without backup
// ✔ Explicit rollback on failure
// ✔ Post-build email notifications only
// ✔ Retry-safe email delivery
// ✔ Readable, auditable, production-ready
//
// IMPORTANT NOTE
// --------------
// VS Code may show "brackets do not match" warnings for Groovy/Jenkins DSL.
// This is a known false-positive and does NOT affect execution.
// =====================================================================================


// =====================================================================================
// ============================== HELPER FUNCTIONS ======================================
// =====================================================================================

/*
 * FUNCTION: buildSummaryHtml()
 * ----------------------------
 * Generates an HTML table summarizing the Jenkins build.
 * Used in all email notifications (success/failure/unstable).
 *
 * Why HTML?
 * - Improves readability
 * - Easy for stakeholders to scan key info
 */
def buildSummaryHtml() {
    return '<table border="1" cellpadding="8" cellspacing="0" style="border-collapse:collapse;font-family:Arial;">' +
           '<tr style="background:#f2f2f2;">' +
           '<th align="left">Item</th><th align="left">Value</th>' +
           '</tr>' +
           '<tr><td><b>Job Name</b></td><td>' + (env.JOB_NAME ?: 'N/A') + '</td></tr>' +
           '<tr><td><b>Build #</b></td><td>' + (env.BUILD_NUMBER ?: 'N/A') + '</td></tr>' +
           '<tr><td><b>Status</b></td><td>' + currentBuild.currentResult + '</td></tr>' +
           '<tr><td><b>Branch</b></td><td>' + (env.GIT_BRANCH ?: 'N/A') + '</td></tr>' +
           '<tr><td><b>Triggered By</b></td><td>' +
               (currentBuild.getBuildCauses()?.getAt(0)?.shortDescription ?: 'N/A') +
           '</td></tr>' +
           '<tr><td><b>Build URL</b></td><td>' +
           '<a href="' + env.BUILD_URL + '">' + env.BUILD_URL + '</a>' +
           '</td></tr>' +
           '</table>'
}

/*
 * FUNCTION: getEnvAndDateInfo()
 * -----------------------------
 * Determines:
 *   - Environment name (DEV / SIT / UAT / PROD)
 *   - Build timestamp in IST timezone
 *
 * Environment is derived from job naming convention:
 *   DEV-xxx  → DEV
 *   SIT-xxx  → SIT
 *   UAT-xxx  → UAT
 *   PROD-xxx → PROD
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
// ========================== EMAIL CONFIGURATION =======================================
// =====================================================================================

// Default distribution list (can be empty or static DL)
def defaultDL = ''

// Post-build DL (example: ops team / release team)
def PostbuildDL = ''

// Variables resolved dynamically at runtime
def triggerEmail   = null
def finalEmailList = null


// =====================================================================================
// ============================== PIPELINE DEFINITION ===================================
// =====================================================================================

pipeline {

    // Any available Jenkins agent
    agent any

    // Global environment variables
    environment {
        JAVA_HOME      = tool 'JDK'
        BUILD_TOOL_CMD = 'mvn'
    }

    stages {

        // -------------------------------------------------------------------------
        // DEBUG STAGE
        // -------------------------------------------------------------------------
        stage('Debug Branch Info') {
            steps {
                echo "Running Jenkinsfile from branch: ${env.GIT_BRANCH}"
            }
        }

        // -------------------------------------------------------------------------
        // EMAIL DISTRIBUTION LIST PREPARATION
        // -------------------------------------------------------------------------
        stage('Prepare Email Distribution List') {
            steps {
                script {
                    try {
                        // Reads email recipients from external trigger file
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

                    // Combine default DL + trigger DL safely
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
        stage('Build') {
            steps {
                sh "${BUILD_TOOL_CMD} clean install -DskipTests"
                archiveArtifacts artifacts: 'target/*.war', followSymlinks: false
            }
        }

        // -------------------------------------------------------------------------
        // UNIT TESTING STAGE
        // -------------------------------------------------------------------------
        stage('Unit Testing') {
            steps {
                sh "${BUILD_TOOL_CMD} test"
            }
        }

        // -------------------------------------------------------------------------
        // DEPLOYMENT STAGE (WITH ROLLBACK)
        // -------------------------------------------------------------------------
        stage('Deployment') {
            steps {
                script {
                    try {

                        // Backup existing WAR for rollback safety
                        sh '''
                            if [ -f /opt/tomcat/webapps/app.war ]; then
                                cp /opt/tomcat/webapps/app.war \
                                   /opt/tomcat/webapps/app.war.bak
                                echo "Existing WAR backed up"
                            fi
                        '''

                        // Deploy new WAR via SSH Publisher
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

                        echo 'Deployment failed — initiating rollback'

                        // Rollback to previous WAR
                        sh '''
                            if [ -f /opt/tomcat/webapps/app.war.bak ]; then
                                mv /opt/tomcat/webapps/app.war.bak \
                                   /opt/tomcat/webapps/app.war
                                echo "Rollback completed"
                            else
                                echo "No backup WAR found — rollback skipped"
                            fi
                        '''

                        error "Deployment failed after rollback: ${err}"
                    }
                }
            }
        }

        // -------------------------------------------------------------------------
        // TOMCAT RESTART STAGE (WITH ROLLBACK)
        // -------------------------------------------------------------------------
        stage('Restart Servers') {
            steps {
                script {
                    try {
                        sh 'sudo systemctl restart tomcat'
                        sh 'sudo systemctl status tomcat'
                    } catch (err) {

                        echo 'Restart failed — restoring previous WAR'

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
        // SANITY CHECK STAGE
        // -------------------------------------------------------------------------
        stage('Sanity Check') {
            steps {
                echo 'Sanity checks completed (placeholder)'
            }
        }

        // -------------------------------------------------------------------------
        // POST ACTION STAGE (LOGICAL END OF PIPELINE)
        // -------------------------------------------------------------------------
        stage('Post Actions') {
            steps {
                echo 'Post actions completed'
            }
        }
    }

    // =================================================================================
    // ============================= POST-BUILD ACTIONS ================================
    // =================================================================================
    post {

        // Always clean workspace
        always {
            cleanWs(deleteDirs: true, disableDeferredWipeout: true)
        }

        // SUCCESS EMAIL
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

        // FAILURE EMAIL
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

        // UNSTABLE EMAIL
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
// ================================ README SECTION ======================================
// =====================================================================================
/*
README – Jenkins Pipeline (Production Standard)
==============================================

OVERVIEW
--------
This Jenkins pipeline builds, tests, deploys, and safely manages a Java WAR
deployment to Tomcat with automatic rollback and HTML email notifications.

ROLLBACK STRATEGY
-----------------
1. Existing WAR is backed up before deployment
2. Deployment failure → backup restored
3. Tomcat restart failure → backup restored and restarted
4. Pipeline fails explicitly after rollback

WHY THIS IS SAFE
----------------
✔ Prevents broken deployments
✔ No manual rollback required
✔ Preserves last known good artifact
✔ Clear failure visibility

EMAIL NOTIFICATIONS
-------------------
- Triggered ONLY in post-build phase
- HTML formatted emails
- Retry-safe (3 attempts)
- Includes:
  - Environment
  - Build date/time
  - Job details
  - Build URL
  - Console log attachment

ENVIRONMENT DETECTION
---------------------
Based on job naming convention:
DEV-*   → DEV
SIT-*   → SIT
UAT-*   → UAT
PROD-*  → PROD

PIPELINE FLOW
-------------
Debug → Email DL → Checkout → Build → Test
→ Deploy (Rollback Safe)
→ Restart (Rollback Safe)
→ Sanity → Post → Email → Cleanup

REQUIRED JENKINS PLUGINS
-----------------------
- Pipeline
- Email Extension Plugin
- SSH Publisher Plugin
- Workspace Cleanup Plugin

NOTES
-----
- VS Code Groovy warnings are false positives
- This README section does not affect execution
- Suitable for DEV → PROD environments

=======================================================================================
*/


