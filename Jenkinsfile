

import javax.mail.*
import javax.mail.internet.*
import jenkins.model.Jenkins

def sendMail(host, sender, receivers, subject, text) {
    Properties props = System.getProperties()
    props.put("smtp.gmail.com", host)
    Session session = Session.getDefaultInstance(props, null)

    MimeMessage message = new MimeMessage(session)
    message.setFrom(new InternetAddress(sender))
    receivers.split(',').each 
    {
        message.addRecipient(Message.RecipientType.TO, new InternetAddress(it))
    }

    message.setSubject(subject)
    message.setText(text)

    println 'Sending mail to ' + receivers + '.'
    Transport.send(message)
    println 'Mail sent.'
}

// Jive Formater

import java.lang.reflect.Array;
import java.util.regex.*
/**
 * This is a Groovy class to allow easy formatting for Jive social networking systems. 
 * It was designed to work in the email-ext plugin for Jenkins. It should be called from the Pre-send Script area. 
 * Also, it doesn't seem like Jive supports text with multiple formats, so only call one formatting method per block of text.
 * Either formatLine or formatText can and should be called on every line of text that will be sent to the Jive system 
 * prior to calling formatting methods like color or size.
 * <p>
 * The following lines should be added to the Pre-send Script area prior to attempting to invoke any functions.
 * <p>
 * File sourceFile = new File("/your/preferred/path/jive-formatter.groovy");
 * Class groovyClass = new GroovyClassLoader(getClass().getClassLoader()).parseClass(sourceFile);
 * GroovyObject jiveFormatter = (GroovyObject) groovyClass.newInstance();
 * <p>
 * @author Daniel Barker
 */
class JiveFormatter {

    /**
     * Removes special characters from a line of text to prevent errors when emailed to Jive
     * @param line This is the String which will be formatted.
     * @return The line with all Jive special characters removed
     */
    String formatLine(String line) {

        if (line.trim().matches(Pattern.compile("^\\d{2}:\\d{2}:\\d{2}\$"))) { // This pattern matches a line that only contains a timestamp
            line = ""
        } else {
            // These characters are only used in Jive to represent some special reference contained inside.
            // If Jive doesn't recognize what's inside, then it will insert the subject of the discussion.
            def braceMatcher = (line =~ /\[/)
            line = braceMatcher.replaceAll("#")
            braceMatcher = (line =~ /\]/)
            line = braceMatcher.replaceAll("#")

            // These characters are only used in Jive to represent some special reference contained inside.
            // If Jive doesn't recognize what's inside, then it will insert the subject of the discussion.
            def curlyBraceMatcher = (line =~ /\{/)
            line = curlyBraceMatcher.replaceAll("")
            curlyBraceMatcher = (line =~ /\}/)
            line = curlyBraceMatcher.replaceAll("")

            // Dashes in Jive represent a line break when there are three or more together. This will cause Jive 
            // to discard any text following them. When dashes are in pairs on the ends of words, then they represent a strikethrough.
            // Dashes are removed when there are two together to be on the safe side to avoid losing content.
            line = line.replace("---", "===").replaceAll("===-+", "====").replace("--", "==")

            // Underscores in Jive represent a line break when there are five or more together. This will cause Jive
            // to discard any text following them. Underscores are removed when there are five together to be on the safe side to avoid losing content.
            line = line.replace("_____", "=====").replaceAll("=====_+", "=======")

            // Jive sees angle brackets as literal tags when they are surrounding a word or statement, but they must be touching the word or statement.
            // Therefore, we remove any angle brackets surrounding text if it is touching the text.
            line = line.replaceAll("<\\b.*\\b>", { it[0].minus(Pattern.compile(/<|>/)) })
        }

        return line
    }

    /**
     * Removes special characters from a line of text to prevent errors when emailed to Jive. 
     * This is basically the same as formatLine, but added for a case where there is an issue with newLine characters.
     * @param text This is the String which will be formatted.
     * @return The line with all Jive special characters removed
     */
    String formatText(String text) {
        def formattedText = ""
        text.each() { line ->
            formattedText.plus(formatLine(line))
        }

        return formattedText
    }

    /**
     * Puts input text within code formatting for the specified language.
     * @param text The text that will be formatted as code
     * @param type The language for the code formatting
     * @return The formatted text
     */
    String code(String text, String type) {
        return "{code:" + formatLine(type).toLowerCase() + "}" + text + "{code}"
    }

    /**
     * Bolds input text.
     * @param text The text that will be bolded
     * @return The bolded text
     */
    String bold(String text) {
        return "*" + text + "*"
    }

    /**
     * Italicizes input text.
     * @param text The text that will be italicized
     * @return The italicized text
     */
    String italic(String text) {
        return "+" + text + "+"
    }

    /**
     * Underlines input text.
     * @param text The text that will be underlined
     * @return The underlined text
     */
    String underline(String text) {
        return "_" + text + "_"
    }

    /**
     * Superscripts input text.
     * @param text The text that will be superscripted
     * @return The superscripted text
     */
    String superscript(String text) {
        return "^" + text + "^"
    }

    /**
     * Subscripts input text.
     * @param text The text that will be subscripted
     * @return The subscripted text
     */
    String subscript(String text) {
        return "~" + text + "~"
    }

    /**
     * Strikes through input text.
     * @param text The text that will be struck through
     * @return The struck through text
     */
    String strikethrough(String text) {
        return "--" + text + "--"
    }

    /**
     * Puts input text into a numbered list format. The list must be delimited.
     * @param text The text that will be formatted as a numbered list
     * @param delimiter The delimiter on which to base the numbered lines
     * @return The numbered list based on the delimiter
     */
    String numberedList(String text, String delimiter) {
        def numberedText = ""
        text.split(delimiter).each { line -> numberedText = numberedText + "# " + line + "\n"}
        return numberedText
    }

    /**
     * Puts input text into a bulleted list format. The list must be delimited.
     * @param text The text that will be formatted as a bulleted list
     * @param delimiter The delimiter on which to base the numbered lines
     * @return The bulleted list based on the delimiter
     */
    String bulletedList(String text, String delimiter) {
        def bulletedText = ""
        text.split(delimiter).each { line -> bulletedText = bulletedText + "* " + line + "\n"}
        return bulletedText
    }

    /**
     * Places input text within a block quote. Should not contain newLine characters.
     * @param text The text that will be formatted as a block quote
     * @return The block quote formatted text
     */
    String blockQuote(String text) {
        return "bq. " + text
    }

    /**
     * Allows for the use of these symbols: {} []. These are normally not allowed in Jive.
     * @param text The text that will be protected from Jive
     * @return The protected text
     */
    String noFormat(String text) {
        return "{noformat}" + text + "{noformat}"
    }

    /**
     * Colors the input text with the specified color.
     * @param text The text that will be colored
     * @param color The color of the text which can be human readable like red, green, blue, etc. or 
     *              hexadecimal like #fff, #f00, #000, etc.
     * @return The colored text
     */
    String color(String text, String color) {
        return "{color:" + formatLine(color).toLowerCase() + "}" + text + "{color}"
    }

    /**
     * Formats the input text with the specified font style
     * @param text The text that will be styled
     * @param style The font style in plain language
     * @return The styled text
     */
    String font(String text, String style) {
        return "{font:" + formatLine(style).toLowerCase() + "}" + text + "{font}"
    }

    /**
     * Formats the input text with the specified font size
     * @param text The text that will be sized
     * @param size The font size in digits
     * @return The sized text
     */
    String size(String text, String size) {
        return "{size:" + formatLine(size) + "px" + "}" + text + "{size}"
    }

    /**
     * Applies a heading attribute to the input text.
     * @param text The text that will be formatted as a heading
     * @param level The heading level. Jive only recognizes 1 through 6. Anything outside this range 
     *              defaults to the closest recognized level.
     * @return The heading formatted text
     */
    String heading(String text, String level) {
        return "h" + checkHeadingSize(level) + ". " + text
    }

    /**
     * Ensures that a recognized heading level is returned.
     * @param level The level that will be checked
     * @return The returned level that will be recognized in Jive
     */
    private String checkHeadingSize(String level) {
        if (!level.isInteger()) {
            return "1"
        }

        if (level.toInteger() < 1) {
            return "1"
        } else if (level.toInteger() > 6) {
            return "6"
        } else {
            return level
        }
    }

    /**
     * Formats the input text as a quote
     * @param text The text that will be formatted as a quote
     * @return The quote formatted text
     */
    String quote(String text) {
        return "{quote}" + text + "{quote}"
    }

    /**
     * Puts input text within quote formatting with a specified title.
     * @param text The text that will be formatted as a quote
     * @param title The title for the quote block. It appears at the top.
     * @return The quote formatted text with title
     */
    String quoteWithTitle(String text, String title) {
        return "{quote:title=" + formatLine(title) + "}" + text + "{quote}"
    }
}

// end jive Formater


pipeline 
{ 
    agent any
    
    tools {
        maven 'mvn'
    }
    
    options {
        timeout(50)
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

           
           stage('Notifications : Send Email') {
             steps {
                script {
                    def mailRecipients = 'loganathr21@gmail.com'
                    def jobName = currentBuild.fullDisplayName
                    // emailext body: '''${SCRIPT, template="groovy-html.template"}''',
                    emailext body: '''${SCRIPT, template="t2.template"}''',
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
