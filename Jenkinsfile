def mvnCmd = "mvn -s configuration/cicd-settings-nexus3.xml"

          pipeline {
            agent {
              label 'maven'
            }
            stages {
              stage('Build App') {
                steps {
                  git branch: 'eap-7', url: 'http://gogs:3000/gogs/openshift-tasks.git'
                  sh "${mvnCmd} install -DskipTests=true"
                }
              }
              stage('Test') {
                steps {
                  sh "${mvnCmd} test"
                  step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
                }
              }
              stage('Code Analysis') {
                steps {
                  script {
                    sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube:9000 -DskipTests=true"
                  }
                }
              }
              stage('Archive App') {
                steps {
                  sh "${mvnCmd} deploy -DskipTests=true -P nexus3"
                }
              }
              stage('Build Image') {
                steps {
                  sh "cp target/openshift-tasks.war target/ROOT.war"
                  script {
                    openshift.withCluster() {
                      openshift.withProject(env.DEV_PROJECT) {
                        openshift.selector("bc", "tasks").startBuild("--from-file=target/ROOT.war", "--wait=true")
                      }
                    }
                  }
                }
              }
              stage('Deploy DEV') {
                steps {
                  script {
                    openshift.withCluster() {
                      openshift.withProject(env.DEV_PROJECT) {
                        openshift.selector("dc", "tasks").rollout().latest();
                      }
                    }
                  }
                }
              }
              stage('Promote to STAGE?') {
                agent {
                  label 'skopeo'
                }
                steps {
                  timeout(time:15, unit:'MINUTES') {
                      input message: "Promote to STAGE?", ok: "Promote"
                  }

                  script {
                    openshift.withCluster() {
                       openshift.tag("${env.DEV_PROJECT}/tasks:latest", "${env.STAGE_PROJECT}/tasks:stage")
                    }
                  }
                }
              }
              stage('Deploy STAGE') {
                steps {
                  script {
                    openshift.withCluster() {
                      openshift.withProject(env.STAGE_PROJECT) {
                        openshift.selector("dc", "tasks").rollout().latest();
                      }
                    }
                  }
                }
              }
            }
          }
