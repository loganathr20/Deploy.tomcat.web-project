

import javax.mail.*
import javax.mail.internet.*

// def manager = "manager" // probably not what you want


def result = manager.build.result
// manager.listener.logger.println "And the result is: ${result}"
def environment = manager.getEnvVars()

echo " Job Name: ${environment.JOB_NAME} "
echo " Build Number: ${environment.BUILD_NUMBER} " 

// def body = "Job Name: ${environment.JOB_NAME} " + System.getProperty("line.separator") + " Build Number: ${environment.BUILD_NUMBER} " + System.getProperty("line.separator") + " Build Status: ${result} " + System.getProperty("line.separator") + " DEPLOYMENT INFORMATION: Check Deployment Console Output at ${environment.BUILD_URL} " + System.getProperty("line.separator") + " Disclaimer: Please do not reply to this email as this is an auto-generated email from Jenkins" 
def subject = " ${environment.JOB_NAME}>> ${environment.BUILD_NUMBER} >> ${result} "

def sendMail(host, sender, receivers, subject, text) {
    Properties props = System.getProperties()
    props.put("smtp.gmail.com", host)
    Session session = Session.getDefaultInstance(props, null)

    MimeMessage message = new MimeMessage(session)
    message.setFrom(new InternetAddress(sender))
    receivers.split(',').each {
        message.addRecipient(Message.RecipientType.TO, new InternetAddress(it))
    }
    message.setSubject(subject)
    message.setText(text)

    println 'Sending mail to ' + receivers + '.'
    Transport.send(message)
    println 'Mail sent.'
}



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
           //  sendMail('smtp.gmail.com', "loganathr21@gmail.comm", "loganathr21@gmail.com", "APPID>>${subject}", "${body}")
        }
        
        success {
            echo "The job is successful"
            // archiveArtifacts artifacts: 'archiveArtifacts artifacts: \'target/*.jar, target/*.war\'', followSymlinks: false, onlyIfSuccessful: true
            //  sendmail('smtp.gmail.com', "loganathr21@gmail.com", "loganathr21@gmail.com", subject:"SUCCESS: ${currentBuild.fullDisplayName}", body: "Yay, we passed."
        }        
    }

}

