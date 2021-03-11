import jenkins.model.Jenkins

def String getFailedBuildHistory() {
    def message = ''
    // Iterate over previous broken builds to find culprits
    def fullName = 'compute-culprits-testrepo'
    // debug pipeline used for testing
    def jobData = Jenkins.instance.getItemByFullName(fullName)
    def lastStableBuild = jobData.getLastStableBuild()
    def lastBuildNumber = jobData.getLastBuild().getNumber() - 1 // We subtract the current executing build from the list
    if (lastStableBuild != null && lastStableBuild.getNumber() != lastBuildNumber) {
        def culpritsSet = new HashSet()
        // UNCOMMENT ME message += "Responsibles:\n"
        // From oldest to newest broken build, since the last sucessful build, find the culprits to notify them
        // The list order represents who is more responsible to fix the build
        for (int buildId = lastStableBuild.getNumber() + 1; buildId <= lastBuildNumber; buildId++) {
            def lastBuildDetails = Jenkins.getInstance().getItemByFullName(fullName).getBuildByNumber(buildId)
            if (lastBuildDetails != null) {
                lastBuildDetails.getCulpritIds().each({ culprit ->
                    if (!culpritsSet.contains(culprit)) {
                        message += "    ${culprit} (build ${lastBuildDetails.getNumber()})\n"
                        culpritsSet.add(culprit)
                    }
                })
            }
        }
    }
    // Complement the message with information from the current executing build
    if (currentBuild.getCurrentResult() != 'SUCCESS') {
        def culprits = currentBuild.changeSets.collectMany({ it.toList().collect({ it.author }) }).unique()
        if (culprits.isEmpty()) {
            // If there is no change log, use the build executor user
            def name = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause').userName
            // UNCOMMENT ME message += "    ${name} (current build ${currentBuild.getId()})"
            message += "${name} <cuz no changelog>"
        } else {
            // If there is change log, use the committer user
            culprits.each({ culprit ->
                // UNCOMMENT ME message += "    ${culprit} (current build ${currentBuild.getId()})"
                message += "${culprit} <from changelog>"
            })
        }
    }
    return message
}

pipeline {
    agent {
            kubernetes {
                yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:4.3-4
    env:
    - name:  JENKINS_URL
      value: http://jenkins.jenkins.svc.cluster.local
  - name: build
    image: ubuntu
    command:
    - sleep
    args:
    - infinity
    env:
    - name: ENV_VAR1
      value: test-value1
  - name: groovy
    image: groovy
    command:
    - sleep
    args:
    - infinity
    env:
    - name: ENV_VAR2
      value: test-value2
'''.stripIndent()
                defaultContainer 'build'
            }
    }
    options {
        timeout(time: 60, unit: 'MINUTES')
    }
    stages {
        stage('Run build steps in containers defined') {
            steps {
                container('build') {
                    sh 'cat /etc/lsb-release'
                    error('Fail build to trigger post build step')
                }
            }
            post {
                success {
                    sh 'echo ${message}'
                }
                unsuccessful {
                    script {
                        def culprits = getFailedBuildHistory()
                        def userIds = slackUserIdsFromCommitters()
                        def userIdsString = userIds.collect { "<@$it>" }.join(' ')
                        slackSend(color: 'danger', message: "${env.JOB_NAME} ${env.BUILD_NUMBER} Failed. \nCommiter(s): $userIdsString \nPossible Culprit(s): $culprits \n(<${env.BUILD_URL}|Open>) ", notifyCommitters: true)
                    }
                }
            }
        }
    }
}
