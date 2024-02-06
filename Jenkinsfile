
import javax.mail.*
import javax.mail.internet.*
import jenkins.model.Jenkins



File sourceFile = new File("/var/lib/jenkins/email-templates/jive-formatter.groovy");
Class groovyClass = new GroovyClassLoader(getClass().getClassLoader()).parseClass(sourceFile);
GroovyObject jiveFormatter = (GroovyObject) groovyClass.newInstance();
                    

pipeline 
{

// feature 2
   
   agent any 
   
   stages 
    {
 
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

         /*  stage('Notifications : Send Email') {
             steps {
                script {
                    def mailRecipients = 'loganathr21@gmail.com'
                    def jobName = currentBuild.fullDisplayName
                    emailext body: '''${SCRIPT, template="groovy-html.template"}''',
                    mimeType: 'text/html',
                    subject: "[Jenkins] ${jobName}",
                    to: "${mailRecipients}",
                    replyTo: "${mailRecipients}",
                    recipientProviders: [[$class: 'CulpritsRecipientProvider']]
                 }
              }
            } */ 

     }  // end of stages

    post 
    {
         always{
             deleteDir()
          }
                
        success {
            // Send email notification for build success
            echo "The job is successful"
        
            /* emailext (
                subject: "Jenkins Build Success: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: "The Jenkins build job ${env.JOB_NAME} has completed successfully.\n\n Commit: ${env.GIT_COMMIT} \n\n Build URL: ${env.BUILD_URL}",
                to: "loganathr21@gmail.com",
            ) */

           script {
                    def mailRecipients = 'loganathr21@gmail.com'
                    def jobName = currentBuild.fullDisplayName
                    
                    // emailext body: "The Jenkins build job ${env.JOB_NAME} has completed successfully.\n\n Commit: ${env.GIT_COMMIT} \n\n Build URL: ${env.BUILD_URL} \n\n "
                    
                    emailext body: '${SCRIPT, template="groovy-html.template"}',
                    mimeType: 'text/html',
                    subject: "[Jenkins] ${jobName}",
                    to: "${mailRecipients}",
                    replyTo: "${mailRecipients}",
                    recipientProviders: [[$class: 'CulpritsRecipientProvider']]
                 }
          }

        failure {
            // Send email notification for build failure
            echo "The job failed"
        
            emailext (
                subject: "Jenkins Build Failure: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: "The Jenkins build job ${env.JOB_NAME} has failed.\n\nCommit: ${env.GIT_COMMIT}\n\nBuild URL: ${env.BUILD_URL}\n\nConsole Output:\n${currentBuild.rawBuild.getLog(100)}",
                to: "loganathr21@gmail.com",
            )
           }

    }  // end of post
 
}  // end of pipeline
