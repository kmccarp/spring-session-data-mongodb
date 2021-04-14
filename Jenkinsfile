pipeline {
	agent none

	triggers {
		pollSCM 'H/10 * * * *'
	}

	options {
		disableConcurrentBuilds()
	}

	stages {
		stage("Test: baseline (jdk8)") {
			agent {
				docker {
					image 'adoptopenjdk/openjdk8:latest'
					args '-v $HOME/.m2:/tmp/jenkins-home/.m2'
				}
			}
			environment {
				ARTIFACTORY = credentials('02bd1690-b54f-4c9f-819d-a77cb7a9822c')
			}
			steps {
				sh "PROFILE=convergence ci/test.sh"
			}
		}
		stage("Test other configurations") {
			parallel {
				stage("Test: spring-next (jdk8)") {
					agent {
						docker {
							image 'adoptopenjdk/openjdk8:latest'
							args '-v $HOME/.m2:/tmp/jenkins-home/.m2'
						}
					}
					environment {
						ARTIFACTORY = credentials('02bd1690-b54f-4c9f-819d-a77cb7a9822c')
					}
					steps {
						sh "PROFILE=spring-next,convergence ci/test.sh"
					}
				}
				stage("Test: baseline (jdk11)") {
					agent {
						docker {
							image 'adoptopenjdk/openjdk11:latest'
							args '-v $HOME/.m2:/tmp/jenkins-home/.m2'
						}
					}
					environment {
						ARTIFACTORY = credentials('02bd1690-b54f-4c9f-819d-a77cb7a9822c')
					}
					steps {
						sh "PROFILE=convergence ci/test.sh"
					}
				}
				stage("Test: spring-next (jdk11)") {
					agent {
						docker {
							image 'adoptopenjdk/openjdk11:latest'
							args '-v $HOME/.m2:/tmp/jenkins-home/.m2'
						}
					}
					environment {
						ARTIFACTORY = credentials('02bd1690-b54f-4c9f-819d-a77cb7a9822c')
					}
					steps {
						sh "PROFILE=spring-next,convergence ci/test.sh"
					}
				}
				stage("Test: baseline (jdk15)") {
					agent {
						docker {
							image 'adoptopenjdk/openjdk15:latest'
							args '-v $HOME/.m2:/tmp/jenkins-home/.m2'
						}
					}
					environment {
						ARTIFACTORY = credentials('02bd1690-b54f-4c9f-819d-a77cb7a9822c')
					}
					steps {
						sh "PROFILE=convergence ci/test.sh"
					}
				}
				stage("Test: spring-next (jdk15)") {
					agent {
						docker {
							image 'adoptopenjdk/openjdk15:latest'
							args '-v $HOME/.m2:/tmp/jenkins-home/.m2'
						}
					}
					environment {
						ARTIFACTORY = credentials('02bd1690-b54f-4c9f-819d-a77cb7a9822c')
					}
					steps {
						sh "PROFILE=spring-next,convergence ci/test.sh"
					}
				}
			}
		}
		stage('Deploy') {
			agent {
				docker {
					image 'springci/spring-session-data-mongodb-openjdk8-with-jq:latest'
					args '-v $HOME/.m2:/tmp/jenkins-home/.m2'
				}
			}
			options { timeout(time: 20, unit: 'MINUTES') }

			environment {
				ARTIFACTORY = credentials('02bd1690-b54f-4c9f-819d-a77cb7a9822c')
				SONATYPE = credentials('oss-login')
				KEYRING = credentials('spring-signing-secring.gpg')
				PASSPHRASE = credentials('spring-gpg-passphrase')
			}

			steps {
				script {
					PROJECT_VERSION = sh(
							script: "ci/version.sh",
							returnStdout: true
					).trim()

					RELEASE_TYPE = 'milestone' // .RC? or .M?

					if (PROJECT_VERSION.endsWith('SNAPSHOT')) {
						RELEASE_TYPE = 'snapshot'
					} else if (PROJECT_VERSION.endsWith('RELEASE')) {
						RELEASE_TYPE = 'release'
					}

					if (RELEASE_TYPE == 'release') {
						sh "PROFILE=central USERNAME=${SONATYPE_USR} PASSWORD=${SONATYPE_PSW} ci/build-and-deploy-to-maven-central.sh ${PROJECT_VERSION}"

						slackSend(
							color: (currentBuild.currentResult == 'SUCCESS') ? 'good' : 'danger',
							channel: '#spring-session-bot',
							message: "@here Spring Session for MongoDB ${PROJECT_VERSION} is staged on Sonatype awaiting closure and release.")

					} else {
						sh "PROFILE=${RELEASE_TYPE} ci/build-and-deploy-to-artifactory.sh"
					}
				}
			}
		}
		stage('Release documentation') {
			when {
				branch 'release-2.3'
			}
			agent {
				docker {
					image 'springci/spring-session-data-mongodb-openjdk8-with-jq:latest'
					args '-v $HOME/.m2:/tmp/jenkins-home/.m2'
				}
			}
			options { timeout(time: 20, unit: 'MINUTES') }

			environment {
				ARTIFACTORY = credentials('02bd1690-b54f-4c9f-819d-a77cb7a9822c')
			}

			steps {
				script {
					sh 'MAVEN_OPTS="-Duser.name=jenkins -Duser.home=/tmp/jenkins-home" ./mvnw -Pdocs ' +
							'-Dartifactory.server=https://repo.spring.io ' +
							"-Dartifactory.username=${ARTIFACTORY_USR} " +
							"-Dartifactory.password=${ARTIFACTORY_PSW} " +
							"-Dartifactory.distribution-repository=temp-private-local " +
							'deploy -B'
				}
			}
		}
	}

	post {
		changed {
			script {
				slackSend(
						color: (currentBuild.currentResult == 'SUCCESS') ? 'good' : 'danger',
						channel: '#spring-session-bot',
						message: "${currentBuild.fullDisplayName} - `${currentBuild.currentResult}`\n${env.BUILD_URL}")
				emailext(
						subject: "[${currentBuild.fullDisplayName}] ${currentBuild.currentResult}",
						mimeType: 'text/html',
						recipientProviders: [[$class: 'CulpritsRecipientProvider'], [$class: 'RequesterRecipientProvider']],
						body: "<a href=\"${env.BUILD_URL}\">${currentBuild.fullDisplayName} is reported as ${currentBuild.currentResult}</a>")
			}
		}
	}
}
