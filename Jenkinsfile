pipeline 
{
    agent any
    
    tools {
        maven 'mvn'
    }
    
    options {
        timeout(10)
        buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '5', numToKeepStr: '5')
    }
    
    stages 
       {
           stage('Configurations ') {
               steps {
                    echo " Prebuild action "
                    echo  ' JAVA_HOME is $JAVA_HOME '
                    echo ' M2_HOME is $M2_HOME '
                    echo ' JENKINS_HOME is $JENKINS_HOME '
                    echo ' ANT_HOME is $ANT_HOME '
               }
           }

           stage('SCM Checkout  ') {
               steps {
                    echo " Prebuild action "
                    echo  ' JAVA_HOME is $JAVA_HOME '
                    echo ' M2_HOME is $M2_HOME '
                    echo ' JENKINS_HOME is $JENKINS_HOME '
                    echo ' ANT_HOME is $ANT_HOME '
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
               }
           }
           
           stage('post-Build') {
               steps {
                    echo " Post build action"
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
        }
        success {
            echo "The job is successful"
        }
    }

}

