// Sample Jenkinsfile. Not guaranteed to work.
pipeline {
    agent none

    options {
        timestamps()
        buildDiscarder(logRotator(daysToKeepStr:'7'))
    }

    environment {
        PIPELINE_NAME = 'gastropub'

        OUTPUT_DIR    = 'tmp/packer/output'

        SNAPSHOT_REPO = 'pkgs-generic-snapshot-sd'
        QA_REPO       = 'pkgs-generic-qa-sd'
        STABLE_REPO   = 'pkgs-generic-stable-sd'
        RELEASED_REPO = 'pkgs-generic-released-sd'

        REPO_PATH = 'myprojects/gastropub'
    }

    stages {
        stage ('Prep') {
            agent any

            steps {
                // Log the environment variables
                sh(script: 'env')

                script {
                    version = sh(returnStdout: true,
                       script: '''
                           VERSION=$(cat VERSION)

                           if [[ "$VERSION" == "master" ]]; then
                               VERSION="${VERSION}-$(git rev-parse --short=7 HEAD)"
                           fi

                           echo $VERSION
                       ''')
                    env.VERSION = version.trim()

                    env.FULL_VERSION = "${env.VERSION}+build.${env.BUILD_ID}"
                }
            }
        }

        stage ('Build Stage'){
            agent any

            steps {
                parallel (
                    virtualbox: {
                        lock(resource: 'GastropubVirtualboxBuild', inversePrecedence: true) {
                            echo "Build stage"

                            ansiColor('xterm'){
                                sh(script: 'script/build')
                            }
                        }
                    }
                )
            }

            post {
                success {
                    stash name: "gastropub", includes: "${OUTPUT_DIR}/**"
                    milestone 200
                }
                failure{
                  slackSend channel: '#gastropub-jenkins',
                            color:   'danger',
                            message: "Build Stage FAILED\n${env.PIPELINE_NAME} :: ${env.BRANCH_NAME} :: ${env.BUILD_ID}\n${env.BUILD_URL}\n@${env.CHANGE_AUTHOR}",
                            tokenCredentialId: 'jenkins_slack_credentials'
                }
            }
        }

        stage ('Publish Stage') {
            agent any

            when {
                expression { env.BRANCH_NAME =~ /release-.*|master/ }
            }

            steps {
                script {
                    unstash "gastropub"

                    // List all files we want to eventually release to customers here
                    def uploadSpec = """{
                      "files": [
                        {
                          "pattern": "${OUTPUT_DIR}/gastropub-*.ova",
                          "target": "${SNAPSHOT_REPO}/${REPO_PATH}/${VERSION}+build.${BUILD_ID}/"
                        }
                        {
                          "pattern": "${OUTPUT_DIR}/gastropub-*.box",
                          "target": "${SNAPSHOT_REPO}/${REPO_PATH}/${VERSION}+build.${BUILD_ID}/"
                        }
                     ]
                    }"""

                    // Publish the build
                    def server = Artifactory.server 'jenkins_artifactory_server'
                    def buildInfo = server.upload(uploadSpec)
                    server.publishBuildInfo(buildInfo)
                    env.BUILD_NAME = buildInfo.name
                }

                milestone 300
            }
        }

        stage ('Integration Stage'){
            agent any

            when {
                expression { env.BRANCH_NAME =~ /release-.*|master/ }
            }

            steps {
                parallel (
                    virtualbox: {
                        lock(resource: 'GastropubVirtualboxIntegration', inversePrecedence: true) {
                            echo "Integration stage"

                            ansiColor('xterm'){
                                sh(script: 'script/tests/run-integration.sh')
                            }
                        }
                    }
                )
            }

            post {
                success {
                    script {
                        def server = Artifactory.server 'jenkins_artifactory_server'

                        def promotionConfig = [
                            'buildName'          : env.BUILD_NAME,
                            'buildNumber'        : env.BUILD_NUMBER,
                            'sourceRepo'         : env.SNAPSHOT_REPO,
                            'targetRepo'         : env.QA_REPO,
                            'status'             : 'QA',
                            'comment'            : env.RUN_DISPLAY_URL,
                            'includeDependencies': true,
                            'copy'               : true,
                            'failFast'           : true
                        ]

                        server.promote promotionConfig
                    }
                }
            }
        }

        stage ('Decision: Deploy Build to Acceptance Test Environment?') {
            agent none

            when {
                expression { env.BRANCH_NAME =~ /release-.*|master/ }
            }

            steps {
                script {
                    milestone 400
                    input message: 'Deploy to Acceptance Test Environment?'
                    milestone 500
                    currentBuild.rawBuild.keepLogs(true)
                }
            }
        }

        stage ('Acceptance Test Stage'){
            agent any

            when {
                expression { env.BRANCH_NAME =~ /release-.*|master/ }
            }

            steps {
                parallel (
                    virtualbox: {
                        lock(resource: 'GastropubVirtualboxAcceptance', inversePrecedence: true) {
                            echo "Acceptance Test Stage"

                            ansiColor('xterm'){
                                sh(script: 'script/tests/run-acceptance-tests.sh')
                            }
                        }
                    }
                )
            }
        }

        stage ('Decision: Promote Build to Stable?') {
            agent none

            when {
                expression { env.BRANCH_NAME =~ /release-.*/ }
            }

            steps {
                script {
                    milestone 600
                    input message: 'Promote this Build to the Stable channel?'
                    milestone 700
                }
            }
        }

        stage ('Promote To Stable'){
            agent any

            when {
                expression { env.BRANCH_NAME =~ /release-.*/ }
            }

            steps {
                echo "Promoting build to Stable channel"

                script {
                    def server = Artifactory.server 'jenkins_artifactory_server'

                    def promotionConfig = [
                        'buildName'          : env.BUILD_NAME,
                        'buildNumber'        : env.BUILD_NUMBER,
                        'sourceRepo'         : env.QA_REPO,
                        'targetRepo'         : env.STABLE_REPO,
                        'status'             : 'stable',
                        'comment'            : env.RUN_DISPLAY_URL,
                        'includeDependencies': true,
                        'copy'               : true,
                        'failFast'           : true
                    ]

                    server.promote promotionConfig
                }
            }
        }

        stage ('Decision: Promote build to the Released channel?') {
            agent none

            when {
                expression { env.BRANCH_NAME =~ /release-.*/ }
            }

            steps {
                script {
                    milestone 800
                    input message: 'Promote build to the Released channel?'
                    milestone 900
                }
            }
        }

        stage ('Release Stage'){
            agent any

            when {
                expression { env.BRANCH_NAME =~ /release-.*/ }
            }

            steps {
                echo "Releasing build"

                script {
                    def server = Artifactory.server 'jenkins_artifactory_server'

                    def promotionConfig = [
                        'buildName'          : env.BUILD_NAME,
                        'buildNumber'        : env.BUILD_NUMBER,
                        'sourceRepo'         : env.STABLE_REPO,
                        'targetRepo'         : env.RELEASED_REPO,
                        'status'             : 'released',
                        'comment'            : env.RUN_DISPLAY_URL,
                        'includeDependencies': true,
                        'copy'               : true,
                        'failFast'           : true
                    ]

                    server.promote promotionConfig
                }

                milestone 1000
            }
        }
    }
    post {
	always {
		script {
			if(currentBuild.result == 'ABORTED'){
				currentBuild.rawBuild.keepLogs(false)
			}
		}
	}
    }
}
