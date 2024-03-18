@Library('gyroidos_ci_common') _

pipeline {
	agent any

    options {
        copyArtifactPermission('shared_lib_test');
    }

	environment {
		YOCTO_VERSION = 'kirkstone'
		BUILDUSER = "${sh(script:'id -u', returnStdout: true).trim()}"
		KVM_GID = "${sh(script:'getent group kvm | cut -d: -f3', returnStdout: true).trim()}"
		GYROID_ARCH = 'x86'
		GYROID_MACHINE = 'genericx86-64'
		PR_BRANCHES = ""
		BUILDTYPE = 'dev'
		BUILD_INSTALLER = 'n'
	}   
   
//	parameters {
//		choice(name: 'GYROID_ARCH', choices: ['x86', 'arm32', 'arm64'], description: 'GyroidOS Target Architecture')
//		choice(name: 'GYROID_MACHINE', choices: ['genericx86-64', 'apalis-imx8', 'raspberrypi3-64', 'raspberrypi2'], description: 'GyroidOS Target Machine (Must be compatible with GYROID_ARCH!)')
//		string(name: 'PR_BRANCHES', defaultValue: '', description: 'Comma separated list of pull request branches (e.g. meta-trustx=PR-177,meta-trustx-nxp=PR-13,gyroidos_build=PR-97)')
//		choice(name: 'BUILD_INSTALLER', choices: ['y', 'n'], description: 'Build the GyroidOS installer (x86 only)')
//	}


	stages {
		stage ('Prepare workspace') {
			agent {
				dockerfile {
					dir '.'
					additionalBuildArgs '--build-arg=BUILDUSER=$BUILDUSER'
					args '--entrypoint=\'\' --env BUILDNODE="${env.NODE_NAME}"'
					reuseNode true
				}
			}
 
			steps {
				sh """
					echo BUILDUSER: ${env.BUILDUSER}

					whoami

					id
				"""

				stepInitWs(manifest: "testmanifest.xml", workspace: "${WORKSPACE}", gyroid_arch: "x86", gyroid_machine: "genericx86-64")
			}
		}



		stage('Source checks + unit tests') {

			// TODO enable
			//when {
			//	expression {
			//		if (! fileExists("trustme/cml")) {
			//			echo "CML sources not available, skipping initial tests"
			//			return false
			//		} else {
			//			echo "CML sources available, performing initial tests"
			//			return true
			//		} 
			//	}
			//}

			when { expression { return false }}

			parallel {
				stage ('Code Format & Style') {
					steps {
						sh 'echo "Entering format test stage"'
						stepFormatCheck("${env.WORKSPACE}")
					}
				}

				/*
				 Intentionally mark the static code analysis stage as skipped
				 We want to show that we are performing static code analysis, but not
				 as part of Jenkins's pipeline.
				*/
				stage('Static Code Analysis') {
					when {
						expression {
							return false
						}
					}

					steps {
						sh label: 'Perform static code analysis', script: '''
							echo "Static Code Analysis is performed using Semmle."
							echo "Please check GitHub's project for results from Semmle's analysis."
						'''
					}
				}

				stage ('Unit tests') {
					agent {
						dockerfile {
							dir '.'
							additionalBuildArgs '--build-arg=BUILDUSER=$BUILDUSER'
							args '--entrypoint=\'\' --env BUILDNODE="${env.NODE_NAME}"'
							reuseNode true
						}
					}
 
					steps {
						sh 'echo "Entering unit test stage"'
						stepUnitTests("${env.WORKSPACE}")
					}
				}
			} // parallel
		} // stage repo + CML checks

		stage ('Build + Test Images') {

			when { expression { return false }}
			// Build images in parallel
			matrix {
				axes {
					axis {
						name 'BUILDTYPE'
							values 'dev', 'production', 'ccmode', 'schsm'
					}
				}

				// TODO adapt dockerfile locations for upstream commit
				stages {
					stage ('Build image') {
						agent {
							dockerfile {
								dir "."
								additionalBuildArgs '--build-arg=BUILDUSER=$BUILDUSER'
								args '--entrypoint=\'\' -v /yocto_mirror/${YOCTO_VERSION}/${GYROID_ARCH}/sources:/source_mirror -v /yocto_mirror/${YOCTO_VERSION}/${GYROID_ARCH}/sstate-cache:/sstate_mirror --env BUILDNODE="${env.NODE_NAME}"'
								reuseNode false
							}
						}

						steps {
							script {
								// TODO env.PR_BRANCHES back to PR_BRANCHES for upstream commit
								if (env.CHANGE_TARGET == null && env.PR_BRANCHES == "" && env.YOCTO_VERSION == env.BRANCH_NAME) {
									// TODO sync_mirrors: y in this case
									stepBuildImage(workspace: "${WORKSPACE}", gyroid_arch: "x86", gyroid_machine: "genericx86-64", buildtype: "${BUILDTYPE}", build_installer: "n", sync_mirrors: "n")
								} else {
									stepBuildImage(workspace: "${WORKSPACE}", gyroid_arch: "x86", gyroid_machine: "genericx86-64", buildtype: "${BUILDTYPE}", build_installer: "n", sync_mirrors: "n")
								}
							}
						}
					}
				}
			}
		} // stage 'Build + Test Images'

		stage ('Integration Tests') {
				agent {
					dockerfile {
						dir "."
						additionalBuildArgs '--build-arg=BUILDUSER=$BUILDUSER'
						args '--entrypoint=\'\' -v /yocto_mirror:/yocto_mirror --device=/dev/kvm --group-add=$KVM_GID -p 2222 -p 5901 --env BUILDNODE="${env.NODE_NAME}"'
						label 'worker'
					}
				}


			when { expression { return false }}

				steps {
					sh "echo ${env.BUILDNODE}"
					sh "ls /yocto_mirror"
					stepIntegrationTest(workspace: "${WORKSPACE}", buildtype: "${BUILDTYPE}", schsm_serial: "", schsm_pin: "")
				}
		} // stage 'Integration Tests'

		stage ('Token Tests') {
			agent {
				node { label 'testing' }
			}

			steps {
				sh "echo ${env.BUILDNODE}"
				sh "ls /yocto_mirror"
				stepIntegrationTest(workspace: "${WORKSPACE}", buildtype: "${BUILDTYPE}", schsm_serial: "${env.PHYSHSM}", schsm_pin: "12345678")
			}
		} // stage 'Token Tests'
		
	} // stages
} // pipeline
