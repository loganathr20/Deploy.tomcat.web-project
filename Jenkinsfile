// Jenkins Pipeline script for a Java project
// Purpose: Build, test, deploy a Java application, send email notifications with HTML summary

// ===================== Helper Functions =====================

// Function to generate HTML build summary table for email
// Returns a string containing HTML table with job details
def buildSummaryHtml() {
    return '<table border="1" cellpadding="8" cellspacing="0" style="border-collapse:collapse;font-family:Arial;">' +
           '<tr style="background:#f2f2f2;"><th align="left">Item</th><th align="left">Value</th></tr>' +
           '<tr><td><b>Job Name</b></td><td>' + env.JOB_NAME + '</td></tr>' +
           '<tr><td><b>Build #</b></td><td>' + env.BUILD_NUMBER + '</td></tr>' +
           '<tr><td><b>Status</b></td><td>' + currentBuild.currentResult + '</td></tr>' +
           '<tr><td><b>Branch</b></td><td>' + (env.GIT_BRANCH ?: 'N/A') + '</td></tr>' +
           '<tr><td><b>Triggered By</b></td><td>' + currentBuild.getBuildCauses()[0].shortDescription + '</td></tr>' +
           '<tr><td><b>Build URL</b></td><td><a href="' + env.BUILD_URL + '">' + env.BUILD_URL + '</a></td></tr>' +
           '</table>'
}

// Function to determine environment name and current build date/time
// Environment is inferred from job name prefix (DEV-, SIT-, UAT-, PROD-)
def getEnvAndDateInfo() {
    def name = env.JOB_NAME?.toUpperCase() // Convert job name to uppercase for uniformity
    def envName = 'UNKNOWN'

    if      (name?.startsWith('PROD-')) { envName = 'PROD' }
    else if (name?.startsWith('UAT-'))  { envName = 'UAT' }
    else if (name?.startsWith('SIT-'))  { envName = 'SIT' }
    else if (name?.startsWith('DEV-'))  { envName = 'DEV' }

    // Get current date/time in IST timezone
    def buildDate = new Date().format('dd-MMM-yyyy HH:mm:ss', TimeZone.getTimeZone('Asia/Kolkata'))

    return [envName: envName, buildDate: buildDate]
}

// ===================== Email Distribution Lists =====================
// Default email distribution list (used if trigger file is missing)
def defaultDL = ''
// Post-build email list (additional)
def PostbuildDL = ''

// Variables to hold emails read from trigger file and final list
def triggerEmail = null
def finalEmailList = null

// ===================== Pipeline Definition =====================
pipeline {
    agent any // Run on any available agent

    environment {
        JAVA_HOME = tool 'JDK'   // Java home configured in Jenkins tools
        BUILD_TOOL_CMD = 'mvn'   // Build command (Maven)
    }

    stages {

        // ------------------- Stage 1: Debug Branch Info -------------------
        stage('Debug Branch Info') {
            steps {
                echo "Running Jenkinsfile from branch: ${env.GIT_BRANCH}"
                // Useful for debugging which branch triggered the pipeline
            }
        }

        // ------------------- Stage 2: Prepare Email Distribution List -------------------
        stage('Prepare Email Distribution List') {
            steps {
                script {
                    // Read trigger file for Email addresses
                    triggerEmail = readFile('/home/lraja/Github/Lightweight-Automation/Trigger_SITBuild.txt')
                        .readLines()
                        .find { it.trim().startsWith('Email=') }  // Find line starting with Email=
                        ?.split('=',2)[1]                         // Extract value after '='
                        ?.replaceAll('"','')                       // Remove quotes
                        ?.split(',')                               // Split multiple emails
                        ?.collect { it.trim() }                    // Trim spaces
                        ?.join(',')                                // Join as comma-separated string

                    // Combine default DL and trigger email list
                    finalEmailList = [defaultDL, triggerEmail].findAll{ it }.join(',')

                    echo "Final email list resolved as: ${finalEmailList ?: defaultDL}"
                }
            }
        }

        // ------------------- Stage 3: Checkout SCM -------------------
        stage('Checkout SCM') {
            steps {
                script {
                    checkout scm // Checkout source code from configured SCM
                    echo 'Source code checked out successfully.'
                }
            }
        }

        // ------------------- Stage 4: Build -------------------
        stage('Build') {
            steps {
                script {
                    // Compile and package the Java project
                    sh "${BUILD_TOOL_CMD} clean install -DskipTests"
                    // Archive generated WAR files for reference
                    archiveArtifacts artifacts: 'target/*.war', followSymlinks: false
                }
            }
        }

        // ------------------- Stage 5: Unit Testing -------------------
        stage('Unit Testing') {
            steps {
                script {
                    sh "${BUILD_TOOL_CMD} test" // Run unit tests
                }
            }
        }

        // ------------------- Stage 6: Deployment -------------------
        stage('Deployment') {
            steps {
                script {
                    try {
                        sshPublisher(
                            publishers: [
                                sshPublisherDesc(
                                    configName: 'tomcat', // SSH config in Jenkins
                                    verbose: true,
                                    transfers: [
                                        sshTransfer(
                                            sourceFiles: 'target/*.war',  // Files to transfer
                                            removePrefix:'target/',       // Remove target folder prefix
                                            remoteDirectory:'/opt/tomcat/webapps/' // Destination on server
                                        )
                                    ]
                                )
                            ]
                        )
                        echo 'Application deployed successfully.'
                    } catch(err) {
                        error "Deployment failed: ${err}" // Fail the build if deployment fails
                    }
                }
            }
        }

        // ------------------- Stage 7: Restart Servers -------------------
        stage('Restart Servers') {
            steps {
                script {
                    // Restart Tomcat server to pick up new deployment
                    sh 'sudo systemctl restart tomcat'
                    sh 'sudo systemctl status tomcat'
                }
            }
        }

        // ------------------- Stage 8: Sanity Check -------------------
        stage('Sanity Check') {
            steps {
                script { echo 'Sanity checks completed (placeholder)' }
                // Placeholder for any post-deployment validation
            }
        }

        // ------------------- Stage 9: Post Actions -------------------
        stage('Post Actions') {
            steps {
                script { echo 'Post actions completed' }
                // Placeholder for any additional post-build actions
            }
        }
    }

    // ===================== Post-Build Actions =====================
    post {

        // Always clean workspace after pipeline ends
        always { cleanWs(deleteDirs:true, disableDeferredWipeout:true) }

        // ------------------- Success Notification -------------------
        success {
            script {
                def info = getEnvAndDateInfo()
                retry(3) { // Retry email sending up to 3 times
                    emailext(
                        to: "${PostbuildDL},${finalEmailList}",
                        subject: "[SUCCESS] [${info.envName}] ${env.JOB_NAME} #${env.BUILD_NUMBER} | ${info.buildDate}",
                        mimeType: 'text/html',
                        body: '<h2 style="color:green;">Build Successful ✅</h2>' +
                              '<p><b>Environment:</b> ' + info.envName + '</p>' +
                              '<p><b>Build Date/Time:</b> ' + info.buildDate + '</p>' +
                              buildSummaryHtml(),
                        attachLog: true
                    )
                }
            }
        }

        // ------------------- Failure Notification -------------------
        failure {
            script {
                def info = getEnvAndDateInfo()
                retry(3) {
                    emailext(
                        to: "${PostbuildDL},${finalEmailList}",
                        subject: "[FAILED] [${info.envName}] ${env.JOB_NAME} #${env.BUILD_NUMBER} | ${info.buildDate}",
                        mimeType: 'text/html',
                        body: '<h2 style="color:red;">Build Failed ❌</h2>' +
                              '<p><b>Environment:</b> ' + info.envName + '</p>' +
                              '<p><b>Build Date/Time:</b> ' + info.buildDate + '</p>' +
                              buildSummaryHtml(),
                        attachLog: true
                    )
                }
            }
        }

        // ------------------- Unstable Notification -------------------
        unstable {
            script {
                def info = getEnvAndDateInfo()
                retry(3) {
                    emailext(
                        to: "${PostbuildDL},${finalEmailList}",
                        subject: "[UNSTABLE] [${info.envName}] ${env.JOB_NAME} #${env.BUILD_NUMBER} | ${info.buildDate}",
                        mimeType: 'text/html',
                        body: '<h2 style="color:orange;">Build Unstable ⚠️</h2>' +
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


