def devices = ['x86_64',
               'wrt3200',
               'wrt32x',
               'espressobin']

def jobs = [:] // dynamically populated later on

def credentialsId = 'buildbot'

void buildMFW(String device, String libc, String startClean, String makeOptions, String buildBranch, String toolsDir, String credentialsId) {
  sshagent (credentials:[credentialsId]) {
    sh "docker-compose -f ${toolsDir}/docker-compose.build.yml -p mfw_${device} pull"
    sh "docker-compose -f ${toolsDir}/docker-compose.build.yml -p mfw_${device} run build -d ${device} -l ${libc} -c ${startClean} -m '${makeOptions}' -v ${buildBranch}"
  }
}

void archiveMFW(String device, String toolsDir, String artifactsDir) {
  sh "rm -fr ${artifactsDir}"
  sh "${toolsDir}/version-images.sh -d ${device} -o ${artifactsDir} -c -t \$(cat tmp/version.date)"
}

pipeline {
  agent none

  triggers {
    upstream(upstreamProjects:"packetd/${env.BRANCH_NAME}, sync-settings/${env.BRANCH_NAME}, classd/${env.BRANCH_NAME}, feeds/${env.BRANCH_NAME}, admin/${env.BRANCH_NAME}, bpfgen/${env.BRANCH_NAME}",
             threshold: hudson.model.Result.SUCCESS)
  }

  parameters {
    choice(name:'libc', choices:['musl', 'glibc'], description:'libc to link against')
    string(name:'buildBranch', defaultValue:env.BRANCH_NAME, description:'branch to use for feeds.git and mfw_build.git')
    choice(name:'startClean', choices:['false', 'true'], description:'start clean')
    string(name:'makeOptions', defaultValue:'-j32', description:'options passed directly to make')
  }

  stages {
    stage('Build') {
      steps {
        script {
	  for (device in devices) {
	    def myDevice = "${device}" // FIXME: cmon now
	    jobs[myDevice] = {
	      node('mfw') {
		stage(myDevice) {
		  def buildDir = "${env.HOME}/build-mfw-${env.BRANCH_NAME}-${myDevice}"
                  def toolsDir = "${env.HOME}/tools-mfw-${env.BRANCH_NAME}-${myDevice}"
                  def artifactsDir = "tmp/artifacts"

		  if (env.BRANCH_NAME =~ /^(master|MFW|release-\d)/) {
		     branch = "${buildBranch}"
		  } else { // force master
		     branch = 'master'
                  }

		  dir(toolsDir) { 
                    git url:"git@github.com:untangle/mfw_build", branch:branch, credentialsId:credentialsId
                  }
		  dir(buildDir) {
                    checkout scm

                    buildMFW(myDevice, libc, startClean, makeOptions, branch, toolsDir, credentialsId)

		    if (myDevice == 'x86_64') {
                      stash(name:"rootfs-${myDevice}", includes:"bin/targets/**/*generic-rootfs.tar.gz")
                    }

                    archiveMFW(myDevice, toolsDir, "${env.WORKSPACE}/${artifactsDir}")
                  }
		  archiveArtifacts artifacts:"${artifactsDir}/*", fingerprint:true
		}
              }
	    }
	  }

          parallel jobs
        }
      }

      post {
	changed {
	  script {
	    // set result before pipeline ends, so emailer sees it
	    currentBuild.result = currentBuild.currentResult
          }
          emailext(to:'nfgw-engineering@untangle.com', subject:"${env.JOB_NAME} #${env.BUILD_NUMBER}: ${currentBuild.result}", body:"${env.BUILD_URL}")
          slackSend(channel:"#team_engineering", message:"${env.JOB_NAME} #${env.BUILD_NUMBER}: ${currentBuild.result} at ${env.BUILD_URL}")
	}
      }
    }

    stage('Test') {
      parallel {
        stage('Test x86_64') {
	  agent { label 'mfw' }

          environment {
            device = 'x86_64'
            toolsDir = "${env.HOME}/tools-mfw-${env.BRANCH_NAME}-${device}"
	    rootfsTarballName = 'mfw-x86-64-generic-rootfs.tar.gz'
	    rootfsTarballPath = "bin/targets/x86/64/${rootfsTarballName}"
	    dockerfile = "${toolsDir}/docker-compose.test.yml"
          }

          stages {
            stage('Prep x86_64') {
              steps {
                unstash(name:"rootfs-${device}")
                sh("test -f ${rootfsTarballPath}")
		sh("mv -f ${rootfsTarballPath} ${toolsDir}")
              }
            }

            stage('TCP services') {
              steps {
                dir('mfw') {
                  script {
                    try {
                      sh("docker-compose -f ${dockerfile} build --build-arg ROOTFS_TARBALL=${rootfsTarballName} mfw")
                      sh("docker-compose -f ${dockerfile} up --abort-on-container-exit --exit-code-from test")
                    } catch (exc) {
                      currentBuild.result = 'UNSTABLE'
                      unstable('TCP services test failed')
                    }
                  }
                }
              }
            }
          }
        }
      }

      post {
	changed {
	  script {
	    // set result before pipeline ends, so emailer sees it
	    currentBuild.result = currentBuild.currentResult
          }
          emailext(to:'nfgw-engineering@untangle.com', subject:"${env.JOB_NAME} #${env.BUILD_NUMBER}: ${currentBuild.result}", body:"${env.BUILD_URL}")
          slackSend(channel:"#team_engineering", message:"${env.JOB_NAME} #${env.BUILD_NUMBER}: ${currentBuild.result} at ${env.BUILD_URL}")
	}
      }
    }

  }
}
