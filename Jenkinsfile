

// Jenkins Pipeline script for a Java project
// This script defines a series of stages for building, testing, and deploying a Java application.

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
                    echo 'Starting deployment process...'
                    // *** CUSTOMIZE THIS SECTION ***  OPTION 1
                    // This is a placeholder for your actual deployment logic.
                    // Examples:
                    // - Copy WAR/JAR to application server:
                    // sh 'scp target/dtw-1.0.0.war lraja@LinuxMint-Thinkcentre:~'
                    // sh 'ssh lraja@LinuxMint-Thinkcentre'
                    // 'sudo mv ~/dtw-1.0.0.war /opt/tomcat/webapps/'
                    // sh 'cp target/dtw-1.0.0.war /opt/tomcat/webapps/'
                    // 'scp target/dtw-1.0.0.war lraja@LinuxMint-Thinkcentre:/opt/tomcat/webapps/'

                    // sh 'scp target/dtw-1.0.0.war lraja@LinuxMint-Thinkcentre:/opt/tomcat/webapps/'
                    // - Deploy to a container (e.g., Docker, Kubernetes):
                    //   sh 'docker build -t your-app .'
                    //   sh 'docker push your-registry/your-app'
                    //   kubernetesDeploy() // If using Kubernetes plugin
                    // - Deploy using a specific tool (e.g., Ansible, Chef, Puppet)
                    //   sh 'ansible-playbook deploy.yml'


                    // Option 2: Recommended - Using 'publishOverSsh' (SSH Publisher Plugin)
                    // This plugin simplifies file transfers and command execution over SSH.
                    // Requires "Publish Over SSH" plugin installed and configured in Jenkins global settings.
                    // You need to define SSH Servers under "Manage Jenkins" -> "Configure System" -> "Publish over SSH".
                    // Example:
                     sshPublisher(publishers: [
                         sshPublisherDesc(
                             configName: 'LinuxMint-Thinkcentre', // Name configured in Jenkins global settings
                             transfers: [
                                 sshTransfer(
                                     sourceFiles: 'target/dtw-1.0.0.war',
                                     remoteDirectory: '/opt/tomcat/webapps/',
                                     removePrefix: 'target/'
                                 )
                             ]
                         )
                         echo 'Application deployed. (Placeholder for actual deployment steps)'
                       ]
       
                    }
           }    
	}


   // Stage 5: Restart Servers
        stage('Restart Servers') {
            steps {
                script {
                    echo 'Restarting application servers...'
                    // *** CUSTOMIZE THIS SECTION ***
                    // This is a placeholder for server restart logic.
                    // Examples:
                    // - SSH command to restart a service:
                    //   sh 'ssh user@your-server "sudo systemctl restart your-app-service"'
                    // - Calling a management API or script.
                    // 'ssh lraja@LinuxMint-Thinkcentre "sudo systemctl restart tomcat"'
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
                    // *** CUSTOMIZE THIS SECTION ***
                    // This is a placeholder for post-deployment sanity checks.
                    // Examples:
                    // - Hit a health check endpoint:
                    //   sh 'curl -f http://your-app-url/health || exit 1'
                    // - Check logs for specific messages:
                    //   sh 'ssh user@your-server "grep -q \'Application started\' /var/log/your-app.log"'
                    echo 'Sanity checks completed. Application is up and running. (Placeholder)'
                }
            }
        }

        // Stage 7: Post Actions (e.g., notifications, archiving artifacts)
        stage('Post Actions') {
            steps {
                script {
                    echo 'Executing post-build actions...'
                    // You can archive build artifacts
                    // archiveArtifacts artifacts: 'target/*.jar, target/*.war', fingerprint: true
                    echo 'Post actions completed.'
                }
            }
        }
    }

    // Post-build actions that run after all stages have completed,
    // regardless of whether the build succeeded or failed.
    post {
        // Always runs
        always {
            echo 'Pipeline finished. Cleaning up workspace...'
            // cleanWs() // Cleans up the workspace on the agent
        }
        // Runs only if the build succeeded
        success {
            echo 'Build and deployment succeeded! Sending success notification...'
            // mail to: 'devs@example.com',
            //      subject: "Jenkins Build Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            //      body: "The build for ${env.JOB_NAME} #${env.BUILD_NUMBER} succeeded. Check ${env.BUILD_URL}"
        }
        // Runs only if the build failed
        failure {
            echo 'Build or deployment failed! Sending failure notification...'
            // mail to: 'devs@example.com',
            //      subject: "Jenkins Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            //      body: "The build for ${env.JOB_NAME} #${env.BUILD_NUMBER} failed. Check ${env.BUILD_URL}"
        }
    }
}




