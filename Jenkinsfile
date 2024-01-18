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
           stage('Pre-Build') {
               steps {
                    echo " Hello - Prebuild action"
                    echo  $JAVA_HOME
                    echo $M2_HOME
                    echo $JENKINS_HOME
                    echo $ANT_HOME
               }
           }
           
           stage('Build') {
               steps {
                    sh "mvn clean install"
               }
           }

           stage('post-Build') {
               steps {
                    echo " Hello - Post build action"
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

