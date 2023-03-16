// Validate
// JENKINS_URL=jenkins-jenkins.itzroks-3100015379-raqclh-6ccd7f378ae819553d37d5f2ee142bd6-0000.au-syd.containers.appdomain.cloud
// curl --user "admin:Passw0rd!" -X POST -F "jenkinsfile=Jenkinsfile" https://$JENKINS_URL/pipeline-model-converter/validate

// Image variables
def buildBarImage = "image-registry.openshift-image-registry.svc:5000/jenkins/ace-buildbar:12.0.4.0-ubuntu"
def ocImage = "image-registry.openshift-image-registry.svc:5000/jenkins/oc-deploy:4.10.54"

// Params for Git Checkout-Stage
def gitCp4iDevOpsUtilsRepo = "https://github.com/khongks/cp4i-devops-utils.git"
def gitRepo = "https://github.com/khongks/cp4i-ace-books.git"
def gitDomain = "github.com"

// Params for Build Bar Stage
def appName = "Books"
def barName = "Books"
def projectDir = "cp4i-ace-books"
def utilsDir = "cp4i-devops-utils"

// Params for Deploy Bar Stage
def serverName = "books"
def namespace = "ace"
def configurationList = ""


// ACE dashboard configurations
def aceDashboardHost = "ace-dashboard-dash.ace.svc.cluster.local"
def port = "3443"
def ibmAceSecretName = "ace-dashboard-dash"

// ACE integration server
def aceVersion = "12.0.5.0-r4"
def aceLicense = "L-KSBM-CJ2KWU"
def replicas = "1"

// Artifactory configurations
def artifactoryHost = "artifactory-tools.itzroks-3100015379-raqclh-6ccd7f378ae819553d37d5f2ee142bd6-0000.au-syd.containers.appdomain.cloud"
def artifactoryPort = "443"
def artifactoryRepo = "generic-local"
def artifactoryBasePath = "cp4i"
def artifactoryCredentials = "artifactory_credentials" // defined in Jenkins credentials



pipeline {
    agent none
    stages {
        stage('Build-Bar') {
            agent {
                docker {
                    image "${buildBarImage}"
                    args '-e LICENSE=accept --entrypoint=""'
                }
            }
            steps {
                sh label: '', script: '''#!/bin/bash
                    Xvfb -ac :99 &
                    export DISPLAY=:99
                    export LICENSE=accept
                    pwd
                    echo "This is the build bar stage"
                    ls -all /opt/ibm/ace-12/ace
                    '''
            }
        }
        stage('oc') {
            agent {
                docker { image "${ocImage}"
                args '--entrypoint=""'
                }
            }
            steps {
                sh label: '', script: '''#!/bin/bash
                    echo "This is the oc stage"
                    which oc
                    '''
            }
        }
        stage('git') {
            agent {
                docker { image 'quay.io/ibmgaragecloud/alpine-git'
                args '--entrypoint=""'
                }
            }
            steps {
                sh label: '', script: '''#!/bin/bash
                    echo "This is the git stage"
                    which git
                    '''
            }
        }
    }
}