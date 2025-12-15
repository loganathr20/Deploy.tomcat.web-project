// Jenkins Pipeline script for a Java project
// This script defines a series of stages for building, testing, and deploying a Java application.

/*
def buildSummaryHtml() {
    return """
    <table border="1" cellpadding="8" cellspacing="0" style="border-collapse:collapse;">
        <tr><th align="left">Job Name</th><td>${env.JOB_NAME}</td></tr>
        <tr><th align="left">Build Number</th><td>${env.BUILD_NUMBER}</td></tr>
        <tr><th align="left">Status</th><td>${currentBuild.currentResult}</td></tr>
        <tr><th align="left">Build URL</th>
            <td><a href="${env.BUILD_URL}">${env.BUILD_URL}</a></td>
        </tr>
    </table>
    """
}
*/

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

// email list for default Distribution list. This will send mail additional mail excluding Email configured in Trigger file.
def defaultDL = null
// def defaultDL = 'l_raja@hotmail.com'

// email list for post build action.
def PostbuildDL = null
// def PostbuildDL = 'loganathr21@gmail.com'

// These are populated later (agent/workspace required)
def triggerEmail = null
def finalEmailList = null

pipeline {
    // Agent definition: 'any' means Jenkins will allocate an executor on any available agent.
    // You can specify a label here (e.g., agent { label 'my-java-agent' }) if you have specific agents.
    agent any

    // Environment variables that can be used throughout the pipeline.
    environment {
        // Example: MAVEN_HOME = tool 'Maven 3.8.6' // If using Maven from Jenkins tools configuration
        // You might define paths to build tools, deployment targets, etc.
        JAVA_HOME = tool 'JDK' // Assuming you have a JDK 11 configured in Jenkins Global Tool Configuration
        // Define your build tool command here, e.g., 'mvn' for Maven or 'gradle' for Gradle
        BUILD_TOOL_CMD = 'mvn'
    }

    // Stages define the main steps of your pipeline.
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

                    // Combine default DL + trigger DL
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

        // Stage 1: Checkout SCM (Source Code Management)
        stage('Checkout SCM') {
            steps {
                script {
                    echo 'Checking out source code from SCM...'
                    // 'checkout scm' checks out the code configured in the Jenkins job's SCM settings.
                    // For Git, this would pull the repository.
                    checkout scm
                    echo 'Source code checked out successfully.'
                }
            }
        }

        // Stage 2: Build the Java Project
        stage('Build') {
            steps {
                script {
                    echo 'Starting Java project build...'
                    // Execute the build command.
                    // For Maven: 'mvn clean install -DskipTests' to build without running tests yet.
                    // For Gradle: 'gradle clean build -x test'
                    sh "${BUILD_TOOL_CMD} clean install -DskipTests"
                    archiveArtifacts artifacts: 'target/*.war', followSymlinks: false
                    echo 'Java project built successfully.'
                }
            }
        }

        // Stage 3: Run Unit Tests
        stage('Unit Testing') {
            steps {
                script {
                    echo 'Running unit tests...'
                    // Execute unit tests.
                    // For Maven: 'mvn test' or part of 'mvn install' if not skipped in build stage.
                    // For Gradle: 'gradle test'
                    sh "${BUILD_TOOL_CMD} test"
                    echo 'Unit tests completed.'
                    // You might want to publish test results here, e.g.:
                    // junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        // Stage 4: Deployment
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
                                        sshTransfer(
                                            sourceFiles: 'target/*.war',
                                            removePrefix: 'target/',
                                            remoteDirectory: '/opt/tomcat/webapps/',
                                            execCommand: '''
                                            echo "Restarting Tomcat service..."
                                            systemctl restart tomcat || echo "Tomcat restart failed!"
                                            ''',
                                            execTimeout: 120000
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


        // Stage 5: Restart Servers
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

        // Stage 6: Sanity Check
        stage('Sanity Check') {
            steps {
                script {
                    echo 'Performing sanity checks on the deployed application...'
                    echo 'Sanity checks completed. Application is up and running. (Placeholder)'
                }
            }
        }

        // Stage 7: Post Actions (e.g., notifications, archiving artifacts)
        stage('Post Actions') {
            steps {
                script {
                    echo 'Executing post-build actions...'
                    echo 'Post actions completed.'
                }
            }
        }

        // Stage 8: Email Notification
        /* stage('Email Notification') {
            steps {
                script {
                    def status = currentBuild.currentResult ?: 'SUCCESS'

                    echo "Sending email to: ${finalEmailList}"

                    emailext(
                        to: finalEmailList,
                        subject: "Jenkins Build ${status}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        body: """\
                            Jenkins Build Report

                            Job Name   : ${env.JOB_NAME}
                            Build No   : ${env.BUILD_NUMBER}
                            Status     : ${status}
                            Build URL  : ${env.BUILD_URL}
                        """,
                        mimeType: 'text/plain',
                        attachLog: true
                    )
                }
            }
        } */

    } 


    // Post-build actions that run after all stages have completed
    post {
        always {
            echo 'Pipeline finished. Cleaning up workspace...'
        }

        success {
            echo 'Build and deployment succeeded! Sending success notification...'
            retry(3) {
                emailext(
                    to: "${PostbuildDL},${finalEmailList}",
                    subject: "[SUCCESS] ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    mimeType: 'text/html',
                    body: """
                    <h2 style="color:green;">Build Successful ✅</h2>
                    ${buildSummaryHtml()}
                    """,
                    attachLog: true
                //  compressLog: true
                )
            }
        }

        failure {
            echo 'Build or deployment failed! Sending failure notification...'
            retry(3) {
                emailext(
                    to: "${PostbuildDL},${finalEmailList}",
                    subject: "[FAILED] ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    mimeType: 'text/html',
                    body: """
                    <h2 style="color:red;">Build Failed ❌</h2>
                    ${buildSummaryHtml()}
                    """,
                    attachLog: true
                //  compressLog: true
                )
            }
        }

        unstable {
            echo 'Build unstable! Sending unstable notification...'
            retry(3) {
                emailext(
                    to: "${PostbuildDL},${finalEmailList}",
                    subject: "[UNSTABLE] ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    mimeType: 'text/html',
                    body: """
                    <h2 style="color:orange;">Build Unstable ⚠️</h2>
                    ${buildSummaryHtml()}
                    """,
                    attachLog: true
                //  compressLog: true
                )
            }
        }

    // Cleans up Workspace.
        cleanup {
            echo "Cleaning up workspace..."
            cleanWs(
                deleteDirs: true,
                disableDeferredWipeout: true
            )
        }
    }

}


