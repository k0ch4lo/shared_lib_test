@Library('gyroidos_ci_common') _

pipeline {
    agent any
    
	environment {
	    YOCTO_VERSION = 'kirkstone'
	    BUILDUSER = "${sh(script:'id -u', returnStdout: true).trim()}"
	    KVM_GID = "${sh(script:'getent group kvm | cut -d: -f3', returnStdout: true).trim()}"
	}   

    parameters {
        choice(name: 'GYROID_ARCH', choices: ['x86', 'arm32', 'arm64'], description: 'GyroidOS Target Architecture')
        choice(name: 'GYROID_MACHINE', choices: ['genericx86-64', 'apalis-imx8', 'raspberrypi3-64', 'raspberrypi2'], description: 'GyroidOS Target Machine (Must be compatible with GYROID_ARCH!)')
        string(name: 'PR_BRANCHES', defaultValue: '', description: 'Comma separated list of pull request branches (e.g. meta-trustx=PR-177,meta-trustx-nxp=PR-13,gyroidos_build=PR-97)')
        choice(name: 'BUILD_INSTALLER', choices: ['y', 'n'], description: 'Build the GyroidOS installer (x86 only)')
    }


    stages {
        stage ('Fetch source step') {
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

                stepInitWs(manifest: "testmanifest.xml", workspace: "${WORKSPACE}", gyroid_arch: "${GYROID_ARCH}", gyroid_machine: "${GYROID_MACHINE}")
            }
        }
    }
}

