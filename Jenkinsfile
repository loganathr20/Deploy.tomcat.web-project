pipeline 
{
    agent any
    
    tools {
        maven 'mvn'
    }
    
    options {
        timeout(10)
        buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '2', numToKeepStr: '2')
    }
    
    stages 
       {
           stage('Configurations ') {
               steps {
                    echo " Prebuild action "
                    echo  ' JAVA_HOME is $env.JAVA_HOME '
                    echo ' M2_HOME is $env.M2_HOME '
                    echo ' JENKINS_HOME is $env.JENKINS_HOME '
                    echo ' ANT_HOME is $env.ANT_HOME '
               }
           }
             
           stage('Pre-Build ') {
               steps {
                    echo " Prebuild action "
                    echo  ' JAVA_HOME is $JAVA_HOME '
                    echo ' M2_HOME is $M2_HOME '
                    echo ' JENKINS_HOME is $JENKINS_HOME '
                    echo ' ANT_HOME is $ANT_HOME '
               }
           }
           
           stage('Build ') {
               steps {
                    sh "mvn clean install"
                    archiveArtifacts artifacts: 'target/*.war', followSymlinks: false
               }
           }


          stage('Unit Testing') {
               steps {
                    echo "  Test Phase "
               }
           }

           stage('Deployment') {
               steps {
                    echo "  Deploy Phase "
               }
           }


           stage('Restart Servers') {
               steps {
                    echo "  Restart Servers  Phase "
                   /*  sh '/opt/tomcat/bin/shutdown.sh'
                    sh '/opt/tomcat/bin/startup.sh' */
                    
               }
           }

           
           stage('Sanity Check') {
               steps {
                    echo " Sanity Check action"
               }
           }

           stage('Notification') {
               steps {
                    echo "Email Notification "
               }
           }
       }

    
    post {
        always{
            deleteDir()
        }
        
        failure {
            echo "sendmail -s mvn build failed loganathr21@gmail.com "
        /*    mail to: "loganathr21@gmail.com", subject:"FAILURE: ${currentBuild.fullDisplayName}", body: "Boo, we failed."
             mail to: "loganathr21@gmail.com", subject:"SUCCESS: ${currentBuild.fullDisplayName}", body: "Yay, we passed."
             emailext (
                   subject: "FAILURE Notification : Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                   body: """<p>STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                   <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>""",
                   recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                )
            */
        }
        
        success {
            echo "The job is successful"
            archiveArtifacts artifacts: 'archiveArtifacts artifacts: \'target/*.jar, target/*.war\'', followSymlinks: false, onlyIfSuccessful: true
            
           /*  mail to: "loganathr21@gmail.com", subject:"SUCCESS: ${currentBuild.fullDisplayName}", body: "Yay, we passed."
             emailext (
                   subject: "SUCCESSFUL Notification : Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                   body: """<p>STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                   <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>""",
                   recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                )
            */

        }        
    }

}

