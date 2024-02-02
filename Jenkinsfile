

import javax.mail.*
import javax.mail.internet.*
import jenkins.model.Jenkins



File sourceFile = new File("/var/lib/jenkins/email-templates/jive-formatter.groovy");
Class groovyClass = new GroovyClassLoader(getClass().getClassLoader()).parseClass(sourceFile);
GroovyObject jiveFormatter = (GroovyObject) groovyClass.newInstance();
                    

pipeline 
{ 
    agent any
           
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

           
           stage('Notifications : Send Email') {
             steps {
                script {
                    def mailRecipients = 'loganathr21@gmail.com'
                    def jobName = currentBuild.fullDisplayName
                         
                    emailext body: '''${SCRIPT, template="groovy-html.template"}''',
                    // emailext body: '''${SCRIPT, template="t2.template"}''',
                    mimeType: 'text/html',
                    subject: "[Jenkins] ${jobName}",
                    to: "${mailRecipients}",
                    replyTo: "${mailRecipients}",
                    recipientProviders: [[$class: 'CulpritsRecipientProvider']]
                 }
            }
          }
           
       }  // end of stages
    
    post 
    {
         always{
             deleteDir()
          }
        
        failure {
             echo "sendmail -s mvn build failed loganathr21@gmail.com "
            // send to email
         }
        
        success {
            echo "The job is successful"
            /*  send to email
            emailext (
               subject: "STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
               body: """<p>STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>""",
               recipientProviders: [[$class: 'DevelopersRecipientProvider']] )
            }  */
          }
     } // end of post
 }  // end of pipeline


