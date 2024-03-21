def withMavenWorkspace(Closure body) {
	withMaven(jdk: 'OpenJDK 17 Latest', maven: 'Apache Maven 3.9',
			mavenLocalRepo: env.WORKSPACE_TMP + '/.m2repository',
			options: [
					artifactsPublisher(disabled: true),
					junitPublisher(disabled: true)
			]) {
		withCredentials([string(credentialsId: 'ge.hibernate.org-access-key',
				variable: 'GRADLE_ENTERPRISE_ACCESS_KEY')]) {
			withGradle { // withDevelocity, actually: https://plugins.jenkins.io/gradle/#plugin-content-capturing-build-scans-from-jenkins-pipeline
				body()
			}
		}
	}
}

pipeline {
	agent none
	options {
		buildDiscarder logRotator(daysToKeepStr: '10', numToKeepStr: '3')
		disableConcurrentBuilds(abortPrevious: true)
	}
	stages {
		stage('Default build') {
			agent {
				label 'Worker&&Containers'
			}
			steps {
				timeout(time: 30, unit: 'MINUTES') {
					withMavenWorkspace {
						sh "mvn clean install -U -Pdist -DskipTests"
						dir(env.WORKSPACE_TMP + '/.m2repository') {
							stash name: 'original-build-result', includes: "org/hibernate/infra/"
						}
					}
				}
			}
		}
		stage('Non-default envs') {
			matrix {
				axes {
					axis {
						name 'ENV_NAME'
						values 'elasticsearch-7', 'postgresql', 'opensearch-2.12'
					}
				}
				stages {
					stage('Build') {
						agent {
							label 'Worker&&Containers'
						}
						steps {
							dir(env.WORKSPACE_TMP + '/.m2repository') {
								unstash name: 'original-build-result'
							}
							withMavenWorkspace {
								sh """ \
							mvn clean install -U -Pdependency-update -Pdist -Pci-build -Dscan.tag.${env.ENV_NAME} \
							--fail-at-end \
						"""
							}
						}
					}
				}
			}
		}
	}
}
