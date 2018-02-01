#! groovy
library 'pipeline-library'

timestamps {
  node('osx && npm-publish') {
    def packageVersion = ''
    def isMaster = false
    stage('Checkout') {
      // checkout scm
      // Hack for JENKINS-37658 - see https://support.cloudbees.com/hc/en-us/articles/226122247-How-to-Customize-Checkout-for-Pipeline-Multibranch
      checkout([
        $class: 'GitSCM',
        branches: scm.branches,
        extensions: scm.extensions + [[$class: 'CleanBeforeCheckout']],
        userRemoteConfigs: scm.userRemoteConfigs
      ])

      isMaster = env.BRANCH_NAME.equals('master')
      packageVersion = jsonParse(readFile('package.json'))['version']
      currentBuild.displayName = "#${packageVersion}-${currentBuild.number}"
    }

    nodejs(nodeJSInstallationName: 'node 4.7.3') {
      ansiColor('xterm') {
        timeout(15) {
          stage('Build') {
            // Install yarn if not installed
            if (sh(returnStatus: true, script: 'which yarn') != 0) {
              sh 'npm install -g yarn'
            }
            sh 'yarn install'
            try {
              withEnv(['TRAVIS=true', 'JUNIT_REPORT_PATH=junit_report.xml']) {
                sh 'yarn test'
              }
            } catch (e) {
              throw e
            } finally {
              junit 'junit_report.xml'
            }
            fingerprint 'package.json'
            // Only tag master
            if (isMaster) {
              pushGitTag(name: packageVersion, message: "See ${env.BUILD_URL} for more information.", force: true)
            }
          } // stage
        } // timeout

        stage('Security') {
          // Clean up and install only production dependencies
          sh 'yarn install --production'

          // Scan for NSP and RetireJS warnings
          sh 'yarn global add nsp'
          sh 'nsp check --output summary --warn-only'

          sh 'yarn global add retire'
          sh 'retire --exitwith 0'

          // TODO Run node-check-updates

          step([$class: 'WarningsPublisher', canComputeNew: false, canResolveRelativePaths: false, consoleParsers: [[parserName: 'Node Security Project Vulnerabilities'], [parserName: 'RetireJS']], defaultEncoding: '', excludePattern: '', healthy: '', includePattern: '', messagesPattern: '', unHealthy: ''])
        } // stage

        stage('Publish') {
		  // only publish master and trigger downstream
          if (isMaster) {
            sh 'npm publish'
            // Trigger appc-cli-wrapper job
            build job: 'appc-cli-wrapper', wait: false
          }
        } // stage

        stage('JIRA') {
          if (isMaster) {
            // update affected tickets, create and release version
            updateJIRA('TIMOB', "ioslib ${packageVersion}", scm)
          } // if
        } // stage(JIRA)
      } // ansiColor
    } //nodejs
  } // node
} // timestamps