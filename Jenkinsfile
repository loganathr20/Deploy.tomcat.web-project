// Jenkins Pipeline script for a Java project
// This script defines a series of stages for building, testing, and deploying a Java application.

// START of PIPELINE Customation for Email Notification  and Lightweight Automation 

def buildSummaryHtml() {

    return """
    <table border="1" cellpadding="8" cellspacing="0" style="border-collapse:collapse;font-family:Arial;">
        <tr style="background:#f2f2f2;">
            <th align="left">Item</th>
            <th align="left">Value</th>
        </tr>
        <tr>
            <td><b>Job Name</b></td>
            <td>${env.JOB_NAME}</td>
        </tr>
        <tr>
            <td><b>Build #</b></td>
            <td>${env.BUILD_NUMBER}</td>
        </tr>
        <tr>
            <td><b>Status</b></td>
            <td>${currentBuild.currentResult}</td>
        </tr>
        <tr>
            <td><b>Branch</b></td>
            <td>${env.GIT_BRANCH ?: 'N/A'}</td>
        </tr>
        <tr>
            <td><b>Triggered By</b></td>
            <td>${currentBuild.getBuildCauses()[0].shortDescription}</td>
        </tr>
        <tr>
            <td><b>Build URL</b></td>
            <td><a href="${env.BUILD_URL}">${env.BUILD_URL}</a></td>
        </tr>
    </table>
    """
}

/*
 * ADDITION: Helper function to resolve ENV name and Build Date
 * No existing logic disturbed
 */


def getEnvAndDateInfo() {
    def envName = 'UNKNOWN'

    if (env.JOB_NAME?.toUpperCase().contains('PROD')) { envName = 'PROD' }
    if (env.JOB_NAME?.toUpperCase().contains('UAT'))  { envName = 'UAT' }
    if (env.JOB_NAME?.toUpperCase().contains('SIT'))  { envName = 'SIT' }
    if (env.JOB_NAME?.toUpperCase().contains('DEV'))  { envName = 'DEV' }

    def buildDate = new Date().format('dd-MMM-yyyy HH:mm:ss')

    return [envName: envName, buildDate: buildDate]
}


// email list for default Distribution list. This will send mail additional mail excluding Email configured in Trigger file.
def defaultDL = ''
// def defaultDL = 'l_raja@hotmail.com'

// email list for post build action.
def PostbuildDL = ''
// def PostbuildDL = 'loganathr21@gmail.com'

// These are populated later (agent/workspace required)
def triggerEmail = null
def finalEmailList = null

// END of PIPELINE Customation for Email Notification and Lightweight Automation 

pipeline { 
    
    agent any

    environment {
        JAVA_HOME = tool 'JDK'
        BUILD_TOOL_CMD = 'mvn'
    }

    stages {
        stage('Prepare Email Distribution List') {
            steps {
                script {
                    echo 'Preparing email distribution list from trigger repository...'

                    triggerEmail = readFile('/home/lraja/Github/Lightweight-Automation/Trigger_SITBuild.txt')
                        .readLines()
                        .find { it.trim().startsWith('Email=') }
                        ?.split('=',2)[1]
                        ?.replaceAll('"','')
                        ?.split(',')
                        ?.collect { it.trim() }
                        ?.join(',')

                    finalEmailList = [defaultDL, triggerEmail]
                        .findAll { it }
                        .join(',')

                    if (!triggerEmail) {
                        echo "Trigger email not found. Using default DL only."
                    }

                    echo "Final email list resolved as: ${finalEmailList}"
                }
            }
        }

        stage('Checkout SCM') {
            steps {
                script {
                    echo 'Checking out source code from SCM...'
                    checkout scm
                    echo 'Source code checked out successfully.'
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    echo 'Starting Java project build...'
                    sh "${BUILD_TOOL_CMD} clean install -DskipTests"
                    archiveArtifacts artifacts: 'target/*.war', followSymlinks: false
                    echo 'Java project built successfully.'
                }
            }
        }

        stage('Unit Testing') {
            steps {
                script {
                    echo 'Running unit tests...'
                    sh "${BUILD_TOOL_CMD} test"
                    echo 'Unit tests completed.'
                }
            }
        }

        stage('Deployment') {
            steps {
                script {
                    try {
                        sshPublisher(
                            publishers: [
                                sshPublisherDesc(
                                    configName: 'tomcat',
                                    verbose: true,
                                    transfers: [
                                        sshTransfer (
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
                                error "Deployment failed: ${err}"
                            }
                    }
            }
        }

        stage('Restart Servers') {
            steps {
                script {
                    echo 'Restarting application servers...'
                    sh 'sudo systemctl status tomcat'
                    sh 'sudo systemctl restart tomcat'
                    sh 'sudo systemctl status tomcat'
                    echo 'Servers restarted. (Placeholder for actual restart steps)'
                }
            }
        }

        stage('Sanity Check') {
            steps {
                script {
                    echo 'Performing sanity checks on the deployed application...'
                    echo 'Sanity checks completed. Application is up and running. (Placeholder)'
                }
            }
        }

        stage('Post Actions') {
            steps {
                script {
                    echo 'Executing post-build actions...'
                    echo 'Post actions completed.'
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished. Cleaning up workspace...'
        }

        success {
            script {
                def info = getEnvAndDateInfo()
                retry(3) {
                    emailext(
                        to: "${PostbuildDL},${finalEmailList}",
                        subject: "[SUCCESS] [${info.envName}] ${env.JOB_NAME} #${env.BUILD_NUMBER} | ${info.buildDate}",
                        mimeType: 'text/html',
                        body: """
                        <h2 style="color:green;">Build Successful ✅</h2>
                        <p><b>Environment:</b> ${info.envName}</p>
                        <p><b>Build Date/Time:</b> ${info.buildDate}</p>
                        ${buildSummaryHtml()}
                        """,
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
                        body: """
                        <h2 style="color:red;">Build Failed ❌</h2>
                        <p><b>Environment:</b> ${info.envName}</p>
                        <p><b>Build Date/Time:</b> ${info.buildDate}</p>
                        ${buildSummaryHtml()}
                        """,
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
                        body: """
                        <h2 style="color:orange;">Build Unstable ⚠️</h2>
                        <p><b>Environment:</b> ${info.envName}</p>
                        <p><b>Build Date/Time:</b> ${info.buildDate}</p>
                        ${buildSummaryHtml()}
                        """,
                        attachLog: true
                    )
                }
            }
        }

        cleanup {
            echo "Cleaning up workspace..."
            cleanWs(deleteDirs: true, disableDeferredWipeout: true)
        }
    }
}
 

