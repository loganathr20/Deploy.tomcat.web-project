pipeline 
{
    agent any
    
    tools {
        maven 'Maven363'
    }
    
    options {
        timeout(10)
        buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '5', numToKeepStr: '5')
    }
    
    stages 
       {
           stage('Build') {
               steps {
                    sh "mvn clean install"
               }
           }
       }
    
    post {
        always{
            deleteDir()
        }
        failure {
            echo "sendmail -s mvn build failed receipients@my.com"
        }
        success {
            echo "The job is successful"
        }
    }
}
