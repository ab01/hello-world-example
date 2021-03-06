node('docker_it') {
try {
notifyBuild('STARTED')
stage('Poll') {
checkout([$class: 'GitSCM', branches: [[name: 'master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'Github', url: 'https://github.com/ab01/hello-world-example.git']]])
}
stage('Build'){
sh 'mvn clean verify -DskipITs=true';
}
stage('Static Code Analysis'){
sh 'mvn clean verify sonar:sonar';
}
stage ('Integration Test'){
sh 'mvn clean verify -Dsurefire.skip=true';
}
stage ('Publish to Artifactory'){
def server = Artifactory.server 'Default Artifactory Server'
def uploadSpec = """{
"files": [
{
"pattern": "target/helloworld-example-0.1.0.jar",
"target": "helloworld-greeting-project/${BUILD_NUMBER}/",
"props": "Integration-Tested=Yes;Performance-Tested=No"
}
]
}"""
server.upload(uploadSpec)
}
stash includes: 'target/helloworld-example-0.1.0.jar,src/pt/Hello_World_Test_Plan.jmx', name: 'binary'
} catch (e) {
    // If there was an exception thrown, the build failed
    currentBuild.result = "FAILED"
    throw e
  } finally {
    // Success or failure, always send notifications
    notifyBuild(currentBuild.result)
  }
}
node('docker_pt') {
try {
notifyBuild('STARTED')
stage ('Start Tomcat'){
sh '''cd /home/jenkins/tomcat/bin
./startup.sh''';
}
stage ('Deploy to Testing Env'){
unstash 'binary'
sh 'cp target/helloworld-example-0.1.0.jar /home/jenkins/tomcat/webapps/';
}
stage ('Performance Testing'){
sh '''cd /opt/jmeter/bin/
./jmeter.sh -n -t /home/jenkins/workspace/helloworld-greeting-project/src/pt/Hello_World_Test_Plan.jmx -l /home/jenkins/workspace/helloworld-greeting-project/test_report.jtl''';
step([$class: 'ArtifactArchiver', artifacts: '**/*.jtl'])
}
stage ('Promote build in Artifactory'){
withCredentials([usernameColonPassword(credentialsId: 'artifactory-account', variable:
'credentials')]) {
sh 'curl -u${credentials} -X PUT "http://54.89.156.240:8081/artifactory/api/storage/helloworld-greeting-project/${BUILD_NUMBER}/helloworld-example-0.1.0.jar?properties=Performance-Tested=Yes"';
}
}
} catch (e) {
    // If there was an exception thrown, the build failed
    currentBuild.result = "FAILED"
    throw e
  } finally {
    // Success or failure, always send notifications
    notifyBuild(currentBuild.result)
  }
}

def notifyBuild(String buildStatus = 'STARTED') {
  // build status of null means successful
  buildStatus =  buildStatus ?: 'SUCCESSFUL'

  // Default values
  def colorName = 'RED'
  def colorCode = '#FF0000'
  def subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
  def summary = "${subject} (${env.BUILD_URL})"
  def details = """<p>STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
    <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>"""

  // Override default values based on build status
  if (buildStatus == 'STARTED') {
    color = 'YELLOW'
    colorCode = '#FFFF00'
  } else if (buildStatus == 'SUCCESSFUL') {
    color = 'GREEN'
    colorCode = '#00FF00'
  } else {
    color = 'RED'
    colorCode = '#FF0000'
  }

  // Send notifications
  slackSend baseUrl: 'https://devops-teamx.slack.com/services/hooks/jenkins-ci/', channel: '#jenkins', color: colorCode, message: summary, token: 'GpQIxzcWjo8qrWDUAW3kktTI'

}
