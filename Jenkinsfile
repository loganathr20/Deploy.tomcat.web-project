


import javax.mail.*
import javax.mail.internet.*
// import hudson.model.*

Binding binding = new Binding();
binding.setVariable("manager", manager);

def manager = "my manager" // probably not what you want

/* 
def build = Thread.currentThread().executable
def result = "And the result is: " 
def BUILD_NUMBER = build.number
def JOB_NAME = job.name
def BUILD_URL=build.url  

def body = "Job Name: ${JOB_NAME} " + System.getProperty("line.separator") + " Build Number: ${BUILD_NUMBER} " + System.getProperty("line.separator") + " Build Status: ${result} " + System.getProperty("line.separator") + " DEPLOYMENT INFORMATION: Check Deployment Console Output at ${BUILD_URL} "  + System.getProperty("line.separator") + " Disclaimer: Please do not reply to this email as this is an auto-generated email from Jenkins"
def subject = " ${JOB_NAME}>> ${BUILD_NUMBER} >> ${result} "

*/

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
            // echo "sendmail -s mvn build failed loganathr21@gmail.com "
           //  sendMail('smtp.gmail.com', "loganathr21@gmail.comm", "loganathr21@gmail.com", "APPID>>${subject}", "${body}")

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
            // archiveArtifacts artifacts: 'archiveArtifacts artifacts: \'target/*.jar, target/*.war\'', followSymlinks: false, onlyIfSuccessful: true
            
            // mail to: "loganathr21@gmail.com", subject:"SUCCESS: ${currentBuild.fullDisplayName}", body: "Yay, we passed."

//             sendmail('smtp.gmail.com', "loganathr21@gmail.com", "loganathr21@gmail.com", subject:"SUCCESS: ${currentBuild.fullDisplayName}", body: "Yay, we passed."



         //   sendMail('smtp.gmail.com', "loganathr21@gmail.comm", "loganathr21@gmail.com", "APPID>>${subject}", "${body}")

                      
//             sendMail('mailhost', messageSender, messageReceivers, messageSubject, messageAllText)
/*
             mail to: 'loganathr21@gmail.com',
                 subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) is waiting for input",
                 body: "Please go to ${env.BUILD_URL}."
           
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

