#!/usr/bin/env groovy

/* Tips
1. Keep stages focused on producing one artifact or achieving one goal. This makes stages easier to parallelize or re-structure later.
1. Stages should simply invoke a make target or a self-contained script. Do not write testing logic in this Jenkinsfile.
3. CoreOS does not ship with `make`, so Docker builds still have to use small scripts.
*/

commonCreds = [
  file(credentialsId: 'tectonic-license', variable: 'TF_VAR_tectonic_license_path'),
  file(credentialsId: 'tectonic-pull', variable: 'TF_VAR_tectonic_pull_secret_path'),
  file(credentialsId: 'GCP-APPLICATION', variable: 'GOOGLE_APPLICATION_CREDENTIALS'),
  usernamePassword(
    credentialsId: 'jenkins-log-analyzer-user',
    passwordVariable: 'LOG_ANALYZER_PASSWORD',
    usernameVariable: 'LOG_ANALYZER_USER'
  ),
  [
    $class: 'AzureCredentialsBinding',
    credentialsId: 'azure-tectonic-test-service-principal',
    subscriptionIdVariable: 'ARM_SUBSCRIPTION_ID',
    clientIdVariable: 'ARM_CLIENT_ID',
    clientSecretVariable: 'ARM_CLIENT_SECRET',
    tenantIdVariable: 'ARM_TENANT_ID'
  ],
  [
    $class: 'StringBinding',
    credentialsId: 'github-coreosbot',
    variable: 'GITHUB_CREDENTIALS'
  ]
]

creds = commonCreds.collect()
creds.push(
  [
    $class: 'AmazonWebServicesCredentialsBinding',
    credentialsId: 'tectonic-jenkins-installer'
  ],
)

govcloudCreds = commonCreds.collect()
govcloudCreds.push(
    usernamePassword(
      credentialsId: 'tectonic-jenkins-installer-govcloud',
      passwordVariable: 'AWS_SECRET_ACCESS_KEY',
      usernameVariable: 'AWS_ACCESS_KEY_ID'
    )
)

quayCreds = [
  usernamePassword(
    credentialsId: 'quay-robot',
    passwordVariable: 'QUAY_ROBOT_SECRET',
    usernameVariable: 'QUAY_ROBOT_USERNAME'
  )
]
defaultBuilderImage = 'quay.io/coreos/tectonic-builder:v1.43'
tectonicSmokeTestEnvImage = 'quay.io/coreos/tectonic-smoke-test-env:v5.14'
originalCommitId = 'UNKNOWN'

pipeline {
  agent none
  environment {
    KUBE_CONFORMANCE_IMAGE = 'quay.io/coreos/kube-conformance:v1.8.4_coreos.0'
  }
  options {
    // Individual steps have stricter timeouts. 360 minutes should be never reached.
    timeout(time:6, unit:'HOURS')
    timestamps()
    buildDiscarder(logRotator(numToKeepStr:'20', artifactNumToKeepStr: '20'))
  }
  parameters {
    string(
      name: 'builder_image',
      defaultValue: defaultBuilderImage,
      description: 'tectonic-builder docker image to use for builds'
    )
    string(
      name: 'hyperkube_image',
      defaultValue: '',
      description: 'Hyperkube image. Please define the param like: {hyperkube="<HYPERKUBE_IMAGE>"}'
    )
    booleanParam(
      name: 'RUN_CONFORMANCE_TESTS',
      defaultValue: false,
      description: ''
    )
    booleanParam(
      name: 'RUN_SMOKE_TESTS',
      defaultValue: true,
      description: ''
    )
    booleanParam(
      name: 'RUN_GUI_TESTS',
      defaultValue: true,
      description: ''
    )
    string(
      name: 'COMPONENT_TEST_IMAGES',
      defaultValue: '',
      description: 'List of container images for component tests to run (comma-separated)'
    )
    string(
      name: 'LOGSTASH_BUCKET',
      defaultValue: 'log-analyzer-tectonic-installer',
      description: 's3 bucket name to sync jenkins log files into'
    )
    booleanParam(
      name: 'PLATFORM/AWS',
      defaultValue: true,
      description: ''
    )
    string(
      name: 'aws_region',
      defaultValue: 'us-east-1',
      description: 'aws region to use'
    )
    string(
      name: 'aws_keypair',
      defaultValue: 'tectonic-jenkins',
      description: 'ec2 keypair to use for ssh. must exist in aws_region'
    )
    string(
      name: 'aws_base_domain',
      defaultValue: 'tectonic-ci.de',
      description: 'route53 base domain for tectonic ingress and api'
    )
    booleanParam(
      name: 'PLATFORM/GOVCLOUD',
      defaultValue: true,
      description: ''
    )
    booleanParam(
      name: 'PLATFORM/AZURE',
      defaultValue: true,
      description: ''
    )
    /* Disabled until we start the work again on gcp
    booleanParam(
      name: 'PLATFORM/GCP',
      defaultValue: false,
      description: ''
    )
    */
    booleanParam(
      name: 'PLATFORM/BARE_METAL',
      defaultValue: true,
      description: ''
    )
    string(
      name : 'ghOrg',
      defaultValue: 'coreos',
      description: 'Github organization'
    )
    string(
      name: 'ghRepo',
      defaultValue: 'tectonic-installer',
      description: 'Github repo'
    )
  }

  stages {
    stage('Build & Test') {
      environment {
        GO_PROJECT = '/go/src/github.com/coreos/tectonic-installer'
        MAKEFLAGS = '-j4'
      }
      steps {
        node('worker && ec2') {
          script {
            def err = null
            try {
              timeout(time: 20, unit: 'MINUTES') {
                forcefullyCleanWorkspace()
                checkout scm
                stash name: 'clean-repo', excludes: 'installer/vendor/**,tests/smoke/vendor/**'
                originalCommitId = sh(returnStdout: true, script: 'git rev-parse origin/"\${BRANCH_NAME}"')
                echo "originalCommitId: ${originalCommitId}"

                withDockerContainer(params.builder_image) {
                  ansiColor('xterm') {
                    sh """#!/bin/bash -ex
                    mkdir -p \$(dirname $GO_PROJECT) && ln -sf $WORKSPACE $GO_PROJECT

                    cd $GO_PROJECT/
                    make structure-check
                    make bin/smoke

                    cd $GO_PROJECT/installer
                    make clean
                    make tools
                    make build

                    make dirtycheck
                    make lint
                    make test
                    rm -fr frontend/tests_output
                    """
                    stash name: 'installer-binary', includes: 'installer/bin/linux/installer'
                    stash name: 'node-modules', includes: 'installer/frontend/node_modules/**'
                    stash name: 'smoke-test-binary', includes: 'bin/smoke'
                  }
                }
                withDockerContainer(tectonicSmokeTestEnvImage) {
                  sh"""#!/bin/bash -ex
                    cd tests/rspec
                    rubocop --cache false spec lib
                  """
                }
              }
            } catch (error) {
              err = error
              throw error
            } finally {
              reportStatusToGithub((err == null) ? 'success' : 'failure', 'basic-tests', originalCommitId)
              cleanWs notFailBuild: true
            }
          }
        }
      }
    }

    stage('GUI Tests') {
      when {
        expression {
          return params.RUN_GUI_TESTS
        }
      }
      environment {
        TECTONIC_INSTALLER_ROLE = 'tectonic-installer'
        GRAFITI_DELETER_ROLE = 'grafiti-deleter'
        TF_VAR_tectonic_container_images = "${params.hyperkube_image}"
      }
      steps {
        script {
          def err = null
          try {
            parallel (
              "IntegrationTest AWS Installer Gui": {
                node('worker && ec2') {
                  timeout(time: 10, unit: 'MINUTES') {
                    forcefullyCleanWorkspace()
                    withCredentials(creds) {
                      withDockerContainer(params.builder_image) {
                        ansiColor('xterm') {
                          unstash 'clean-repo'
                          unstash 'installer-binary'
                          unstash 'node-modules'
                          sh """#!/bin/bash -ex
                          cd installer
                          make launch-aws-installer-guitests
                          make gui-aws-tests-cleanup
                          """
                          cleanWs notFailBuild: true
                        }
                      }
                    }
                  }
                }
              },
              "IntegrationTest Baremetal Installer Gui": {
                node('worker && ec2') {
                  timeout(time: 10, unit: 'MINUTES') {
                    forcefullyCleanWorkspace()
                    withCredentials(creds) {
                      withDockerContainer(image: params.builder_image, args: '-u root') {
                        ansiColor('xterm') {
                          unstash 'clean-repo'
                          unstash 'installer-binary'
                          unstash 'node-modules'
                          script {
                            try {
                              sh """#!/bin/bash -ex
                              cd installer
                              make launch-baremetal-installer-guitests
                              """
                            }
                            catch (error) {
                              throw error
                            }
                            finally {
                              sh """#!/bin/bash -x
                              cd installer
                              make gui-baremetal-tests-cleanup
                              make clean
                              """
                              cleanWs notFailBuild: true
                            }
                          }
                        }
                      }
                    }
                  }
                }
              }
            )
          } catch (error) {
            err = error
            throw error
          } finally {
            node('worker && ec2') {
              unstash 'clean-repo'
              reportStatusToGithub((err == null) ? 'success' : 'failure', 'gui-tests', originalCommitId)
            }
          }
        }
      }
    }
    stage("Cluster Live Tests [Smoke,Conformance,Component]") {
      environment {
        TF_VAR_tectonic_container_images = "${params.hyperkube_image}"
        TF_VAR_tectonic_kubelet_debug_config = "--minimum-container-ttl-duration=8h --maximum-dead-containers-per-container=9999 --maximum-dead-containers=9999"
      }

      parallel {
        stage("AWS Cluster Live Tests") {
          when {
            expression {
              return (params.RUN_SMOKE_TESTS || params.RUN_CONFORMANCE_TESTS || params.COMPONENT_TEST_IMAGES != '') && params."PLATFORM/AWS"
            }
          }
          environment {
            TECTONIC_INSTALLER_ROLE = 'tectonic-installer'
            GRAFITI_DELETER_ROLE = 'grafiti-deleter'
            TF_VAR_tectonic_aws_region = "${params.aws_region}"
            TF_VAR_tectonic_aws_ssh_key = "${params.aws_keypair}"
            TF_VAR_tectonic_base_domain = "${params.aws_base_domain}"
            TF_VAR_base_domain = "${params.aws_base_domain}"
          }
          steps {
            script {
              def builds = [:]
              def aws = [
                [file: 'basic_spec.rb', args: ''],
                [file: 'vpc_internal_spec.rb', args: '--device=/dev/net/tun --cap-add=NET_ADMIN -u root'],
                [file: 'network_canal_spec.rb', args: ''],
                [file: 'exp_spec.rb', args: ''],
                [file: 'ca_spec.rb', args: ''],
                [file: 'custom_tls_spec.rb', args: '']
              ]
              aws.each { build ->
                filepath = 'spec/aws/' + build.file
                builds['aws/' + build.file] = runRSpecTest(filepath, build.args, creds)
              }
              parallel builds
            }
          }
        }
        stage("AWS GovCloud Cluster Live Tests") {
          when {
            expression {
              return (params.RUN_SMOKE_TESTS || params.RUN_CONFORMANCE_TESTS || params.COMPONENT_TEST_IMAGES != '') && params."PLATFORM/GOVCLOUD"
            }
          }
          environment {
            TECTONIC_INSTALLER_ROLE = 'tectonic-installer'
            GRAFITI_DELETER_ROLE = 'grafiti-deleter'
          }
          steps {
            script {
              def builds = [:]
              def govcloud = [
                [file: 'vpc_internal_spec.rb', args: '--device=/dev/net/tun --cap-add=NET_ADMIN -u root']
              ]
              govcloud.each { build ->
                filepath = 'spec/govcloud/' + build.file
                builds['govcloud/' + build.file] = runRSpecTest(filepath, build.args, govcloudCreds)
              }
              parallel builds
            }
          }
        }
        stage("Bare Metal Cluster Live Tests") {
          when {
            expression {
              return (params.RUN_SMOKE_TESTS || params.RUN_CONFORMANCE_TESTS || params.COMPONENT_TEST_IMAGES != '') && params."PLATFORM/BARE_METAL"
            }
          }
          steps {
            script {
              def builds = [:]
              def metal = [
                [file: 'basic_spec.rb', args: ''],
                [file: 'custom_tls_spec.rb', args: '']
              ]
              metal.each { build ->
                filepath = 'spec/metal/' + build.file
                builds['metal/' + build.file] = runRSpecTestBareMetal(filepath, creds)
              }
              parallel builds
            }
          }
        }
        stage("Azure Cluster Live Tests") {
          when {
            expression {
              return (params.RUN_SMOKE_TESTS || params.RUN_CONFORMANCE_TESTS || params.COMPONENT_TEST_IMAGES != '') && params."PLATFORM/AZURE"
            }
          }
          steps {
            script {
              def builds = [:]
              def azure = [
                [file: 'basic_spec.rb', args: ''],
                [file: 'private_external_spec.rb', args: '--device=/dev/net/tun --cap-add=NET_ADMIN -u root'],
                /*
                 * Test temporarily disabled
                 [file: 'spec/azure_dns_spec.rb', args: ''],
                 */
                [file: 'external_spec.rb', args: ''],
                [file: 'example_spec.rb', args: ''],
                [file: 'custom_tls_spec.rb', args: '']
              ]
              azure.each { build ->
                filepath = 'spec/azure/' + build.file
                builds['azure/' + build.file] = runRSpecTest(filepath, build.args, creds)
              }
              parallel builds
            }
          }
        }
        stage("GCP Cluster Live Tests") {
          when {
            expression {
              return (params.RUN_SMOKE_TESTS || params.RUN_CONFORMANCE_TESTS || params.COMPONENT_TEST_IMAGES != '') && params."PLATFORM/GCP"
            }
          }
          environment {
            GOOGLE_PROJECT = "tectonic-installer"
          }
          steps {
            script {
              def builds = [:]
              def gcp = [
                /* Disabled until we start the work again on gcp
                 *  [file: 'basic_spec.rb', args: ''],
                 [file: 'ha_spec.rb', args: ''],
                 [file: 'custom_tls_spec.rb', args: '']
                 */
              ]
              gcp.each { build ->
                filepath = 'spec/gcp/' + build.file
                builds['gcp/' + build.file] = runRSpecTest(filepath, build.args, creds)
              }
              parallel builds
            }
          }
        }
      }
    }
    stage('Build docker image')  {
      when {
        branch 'master'
      }
      steps {
        node('worker && ec2') {
          forcefullyCleanWorkspace()
          withCredentials(quayCreds) {
            ansiColor('xterm') {
              unstash 'clean-repo'
              sh """
                docker build -t quay.io/coreos/tectonic-installer:master -f images/tectonic-installer/Dockerfile .
                docker login -u="$QUAY_ROBOT_USERNAME" -p="$QUAY_ROBOT_SECRET" quay.io
                docker push quay.io/coreos/tectonic-installer:master
                docker logout quay.io
              """
              cleanWs notFailBuild: true
            }
          }
        }
      }
    }

  }
  post {
    always {
      node('worker && ec2') {
        forcefullyCleanWorkspace()
        echo "Starting with streaming the logfile to the S3 bucket"
        withDockerContainer(params.builder_image) {
          withCredentials(creds) {
            unstash 'clean-repo'
            sh """#!/bin/bash -xe
            export BUILD_RESULT=${currentBuild.currentResult}
            ./tests/jenkins-jobs/scripts/log-analyzer-copy.sh jenkins-logs
            """
          }
        }
      }
    }
  }
}

def forcefullyCleanWorkspace() {
  return withDockerContainer(
    image: tectonicSmokeTestEnvImage,
    args: '-u root'
  ) {
    ansiColor('xterm') {
      sh """#!/bin/bash -e
        if [ -d "\$WORKSPACE" ]
        then
          rm -rfv \$WORKSPACE/*
        fi
      """
    }
  }
}

def runRSpecTest(testFilePath, dockerArgs, credentials) {
  return {
    node('worker && ec2') {
      def err = null
      try {
        timeout(time: 5, unit: 'HOURS') {
          forcefullyCleanWorkspace()
          ansiColor('xterm') {
            withCredentials(credentials + quayCreds) {
              withDockerContainer(
                image: tectonicSmokeTestEnvImage,
                args: '-u root -v /var/run/docker.sock:/var/run/docker.sock ' + dockerArgs
              ) {
                unstash 'clean-repo'
                unstash 'smoke-test-binary'
                sh """#!/bin/bash -ex
                  mkdir -p templogfiles && chmod 777 templogfiles
                  cd tests/rspec

                  # Directing test output both to stdout as well as a log file
                  rspec ${testFilePath} --format RspecTap::Formatter --format RspecTap::Formatter --out ../../templogfiles/format=tap.log
                """
              }
            }
          }
        }
      } catch (error) {
        err = error
        throw error
      } finally {
        reportStatusToGithub((err == null) ? 'success' : 'failure', testFilePath, originalCommitId)
        archiveArtifacts allowEmptyArchive: true, artifacts: 'build/**/logs/**'
        withDockerContainer(params.builder_image) {
         withCredentials(creds) {
           sh """#!/bin/bash -xe
           ./tests/jenkins-jobs/scripts/log-analyzer-copy.sh smoke-test-logs ${testFilePath}
           """
         }
        }

        cleanWs notFailBuild: true
      }

    }
  }
}

def runRSpecTestBareMetal(testFilePath, credentials) {
  return {
    node('worker && bare-metal') {
      def err = null
      try {
        timeout(time: 5, unit: 'HOURS') {
          ansiColor('xterm') {
            unstash 'clean-repo'
            unstash 'smoke-test-binary'
            withCredentials(credentials + quayCreds) {
              sh """#!/bin/bash -ex
              cd tests/rspec
              export RBENV_ROOT=/usr/local/rbenv
              export PATH="/usr/local/rbenv/bin:$PATH"
              eval \"\$(rbenv init -)\"
              rbenv install -s
              gem install bundler
              bundler install
              bundler exec rspec ${testFilePath} --format RspecTap::Formatter --format RspecTap::Formatter --out ../../templogfiles/format=tap.log
              """
            }
          }
        }
      } catch (error) {
        err = error
        throw error
      } finally {
        reportStatusToGithub((err == null) ? 'success' : 'failure', testFilePath, originalCommitId)
        archiveArtifacts allowEmptyArchive: true, artifacts: 'build/**/logs/**'
        withCredentials(credentials) {
          sh """#!/bin/bash -xe
          ./tests/jenkins-jobs/scripts/log-analyzer-copy.sh smoke-test-logs ${testFilePath}
           """
         }
        cleanWs notFailBuild: true
      }
    }
  }
}

def reportStatusToGithub(status, context, commitId) {
  withCredentials(creds) {
    sh """#!/bin/bash -ex
      GH_ORG=\${ghOrg} GH_REPO=\${ghRepo} ./tests/jenkins-jobs/scripts/report-status-to-github.sh ${status} ${context} ${commitId}
    """
  }
}
