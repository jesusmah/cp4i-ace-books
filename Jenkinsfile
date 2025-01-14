// Image variables
def buildBarImage = "localhost/ace-buildbar:12.0.4.0-ubuntu"
def ocImage = "localhost/oc-deploy:4.10.54-v2"
def gitImage = "quay.io/ibmgaragecloud/alpine-git"

// Params for Git Checkout-Stage
def gitCp4iDevOpsUtilsRepo = "https://github.com/jesusmah/cp4i-devops-utils.git"
def gitAppRepo = "https://github.com/jesusmah/cp4i-ace-books.git"
def gitDomain = "github.com"

// Params for Build Bar Stage
def appName = "Books"
def barName = "Books"
def projectDir = "cp4i-ace-books"
def utilsDir = "cp4i-devops-utils"

// Params for Deploy Bar Stage
def serverName = "books"
def namespace = "ace-jenkins"
def configurationList = ""

// ACE integration server
//def aceVersion = "12.0.5.0-r4"
def aceVersion = "12.0.7.0-r4"
//def aceLicense = "L-KSBM-CJ2KWU"
def aceLicense = "L-APEH-CJUCNR"
def replicas = "1"

// Artifactory configurations
def artifactoryHost = "artifactory-tools.itzroks-66700099q7-0r6md0-6ccd7f378ae819553d37d5f2ee142bd6-0000.us-east.containers.appdomain.cloud"
def artifactoryPort = "443"
def artifactoryRepo = "generic-local"
def artifactoryBasePath = "jenkins"
def artifactoryCredentials = "artifactory_credentials" // defined in Jenkins credentials
def artifactoryNamespace = "tools"



pipeline {
    agent none
    //options {
        // This is required if you want to clean before build
        //skipDefaultCheckout(true)
    //}
    stages {
        stage('Git Checkout') {
            environment {
                GIT_CP4I_DEVOPS_UTILS_REPO = "${gitCp4iDevOpsUtilsRepo}"
                GIT_APP_REPO = "${gitAppRepo}"
                CP4I_DEVOPS_UTILS_DIR = "${utilsDir}"
                PROJECT_DIR = "${projectDir}"
            }
            agent {
                docker { image "${gitImage}"
                args '--entrypoint=""'
                reuseNode true
                }
            }
            steps {
                // Clean before build
                //cleanWs()
                // We need to explicitly checkout from SCM here
                //checkout scm
                sh """
                    # Manual cleanup
                    rm -rf $CP4I_DEVOPS_UTILS_DIR
                    rm -rf $PROJECT_DIR
                    git clone $GIT_CP4I_DEVOPS_UTILS_REPO
                    git clone $GIT_APP_REPO
                    cp -p $CP4I_DEVOPS_UTILS_DIR/templates/integration-server.yaml.tmpl $PROJECT_DIR
                    cp -p $CP4I_DEVOPS_UTILS_DIR/scripts/*.sh $PROJECT_DIR
                    ls -la
                """
            }
        }
        stage('Build Bar File') {
            environment {
                PROJECT_DIR = "${projectDir}"
                BAR_NAME = "${barName}"
                APP_NAME = "${appName}"
            }
            agent {
                docker {
                    image "${buildBarImage}"
                    args '-e LICENSE=accept --entrypoint=""'
                    reuseNode true
                }
            }
            steps {
                sh label: '', script: '''#!/bin/bash
                    Xvfb -ac :99 &
                    export DISPLAY=:99
                    export LICENSE=accept
                    pwd
                    source /opt/ibm/ace-12/server/bin/mqsiprofile
                    cd $PROJECT_DIR
                    BAR_FILE="${BAR_NAME}_${BUILD_NUMBER}.bar"
                    mqsicreatebar -data . -b $BAR_FILE -a $APP_NAME -cleanBuild -trace -configuration . 
                    ls -lha
                    '''
            }
        }
        stage('Upload Bar File') {
            environment {
                PROJECT_DIR = "${projectDir}"
                ARTIFACTORY_HOST = "${artifactoryHost}"
                ARTIFACTORY_REPO = "${artifactoryRepo}"
                ARTIFACTORY_BASE_PATH = "${artifactoryBasePath}"
                BAR_NAME = "${barName}"
                ARTIFACTORY_CREDS = credentials('artifactory_credentials')
            }
            agent {
                docker { image "${buildBarImage}"
                args '--entrypoint=""'
                reuseNode true
                }
            }
            steps {
                sh label: '', script: '''#!/bin/bash
                    set -e
                    cd $PROJECT_DIR
                    ls -lha
                    echo "Calling upload-barfile-to-artifactory.sh ${ARTIFACTORY_HOST} ${ARTIFACTORY_REPO} ${ARTIFACTORY_BASE_PATH} "${BAR_NAME}_${BUILD_NUMBER}.bar" ${ARTIFACTORY_CREDS_USR} ${ARTIFACTORY_CREDS_PSW}"
                    ./upload-barfile-to-artifactory.sh ${ARTIFACTORY_HOST} ${ARTIFACTORY_REPO} ${ARTIFACTORY_BASE_PATH} "${BAR_NAME}_${BUILD_NUMBER}.bar" ${ARTIFACTORY_CREDS_USR} ${ARTIFACTORY_CREDS_PSW}
                    '''
            }
        }
        stage('ACE BarAuth') {
            environment {
                PROJECT_DIR = "${projectDir}"
                NAMESPACE = "${namespace}"
                ARTIFACTORY_NAMESPACE = "${artifactoryNamespace}"
                OC_CREDS = credentials('oc-credentials')
            }
            agent {
                docker { image "${ocImage}"
                args '--entrypoint=""'
                reuseNode true
                }
            }
            steps {
                sh label: '', script: '''#!/bin/bash
                    export KUBECONFIG=$WORKSPACE/.kube/config
                    oc login --token=$OC_CREDS_PSW --server=$OC_CREDS_USR --insecure-skip-tls-verify
                    set -e
                    cd $PROJECT_DIR
                    echo "Check if bar-auth-config ACE Configuration exists"
                    EXISTS=`oc get configuration.appconnect.ibm.com -n ${NAMESPACE} | grep bar-auth-config | wc -l`
                    if [[ $EXISTS -eq 1 ]]
                    then
                        echo "bar-auth-config ACE Configuration exists"
                    else
                        echo "bar-auth-config ACE Configuration does not exists"
                        echo "Getting CA certificate for Artifactory"
                        ARTIFACTORY_ENDPOINT=`oc get route artifactory -n ${ARTIFACTORY_NAMESPACE} -o jsonpath='{.status.ingress[].host}'`
                        # Get certificates from artifactory endpoint
                        openssl s_client -showcerts -verify 5 -connect ${ARTIFACTORY_ENDPOINT}:443 < /dev/null | awk '/BEGIN CERTIFICATE/,/END CERTIFICATE/{ if(/BEGIN CERTIFICATE/){a++}; out="cert"a".pem"; print >out}'
                        # Base64 encode CA certificate
                        CA_ENCODED=`cat cert1.pem | base64 -w0`
                        # Create the secret
                        cat <<EOF | oc apply -f -
                        kind: Secret
                        apiVersion: v1
                        metadata:
                            name: artifactory-cacert-secret
                            namespace: ${NAMESPACE}
                        data:
                            ca.crt: '${CA_ENCODED}'
                            tls.crt: ''
                            tls.key: ''
                        type: kubernetes.io/tls
EOF
        
                        # Create JSON file with Artifactory credentials and CA certificate
                        oc extract secret/artifactory-access -n ${ARTIFACTORY_NAMESPACE}
                        ARTIFACTORY_PASSWORD=`cat ARTIFACTORY_PASSWORD`
                        cat - > bar-auth.json << EOF
                        {
                            "authType":"BASIC_AUTH",
                            "credentials": {
                                "username":"admin",
                                "password":"${ARTIFACTORY_PASSWORD}",
                                "caCertSecret":"artifactory-cacert-secret"
                            }
                        }
EOF
          
                        # Create the secret containing the JSON
                        oc create secret generic bar-auth-secret --from-file=configuration=bar-auth.json --namespace=${NAMESPACE}
                        cat <<EOF | oc apply -f -
                        apiVersion: appconnect.ibm.com/v1beta1
                        kind: Configuration
                        metadata:  
                            name: bar-auth-config  
                            namespace: ${NAMESPACE}
                        spec:  
                            description: Stores bar auth
                            secretName: bar-auth-secret
                            type: barauth
EOF
                    fi
                    '''
                }
        }
        stage('Deploy Intergration Server') {
            environment {
                PROJECT_DIR = "${projectDir}"
                BAR_NAME = "${barName}"
                SERVER_NAME = "${serverName}"
                NAMESPACE = "${namespace}"
                ARTIFACTORY_HOST = "${artifactoryHost}"
                ARTIFACTORY_PORT = "${artifactoryPort}"
                ARTIFACTORY_REPO = "${artifactoryRepo}"
                ARTIFACTORY_BASE_PATH = "${artifactoryBasePath}"
                CONFIGURATION_LIST = "${configurationList}"
                ACE_VERSION = "${aceVersion}"
                ACE_LICENSE = "${aceLicense}"
                REPLICAS = "${replicas}"
                OC_CREDS = credentials('oc-credentials')
            }
            agent {
                docker { image "${ocImage}"
                args '--entrypoint=""'
                reuseNode true
                }
            }
            steps {
                sh label: '', script: '''#!/bin/bash
                    export KUBECONFIG=$WORKSPACE/.kube/config
                    set -e
                    cd $PROJECT_DIR
                    BAR_FILE="${BAR_NAME}_${BUILD_NUMBER}.bar"
                    cat integration-server.yaml.tmpl
                    sed -e "s/{{NAME}}/$SERVER_NAME/g" \
                        -e "s/{{NAMESPACE}}/$NAMESPACE/g" \
                        -e "s/{{ARTIFACTORY_HOST}}/$ARTIFACTORY_HOST/g" \
                        -e "s/{{ARTIFACTORY_PORT}}/$ARTIFACTORY_PORT/g" \
                        -e "s/{{ARTIFACTORY_REPO}}/$ARTIFACTORY_REPO/g" \
                        -e "s/{{ARTIFACTORY_BASE_PATH}}/$ARTIFACTORY_BASE_PATH/g" \
                        -e "s/{{BAR_FILE}}/$BAR_FILE/g" \
                        -e "s/{{CONFIGURATION_LIST}}/$CONFIGURATION_LIST/g" \
                        -e "s/{{ACE_VERSION}}/$ACE_VERSION/g" \
                        -e "s/{{ACE_LICENSE}}/$ACE_LICENSE/g" \
                        -e "s/{{REPLICAS}}/$REPLICAS/g" \
                        integration-server.yaml.tmpl > integration-server.yaml
                    cat integration-server.yaml
                    # oc login --token=$OC_CREDS_PSW --server=$OC_CREDS_USR --insecure-skip-tls-verify
                    oc apply -f integration-server.yaml
                    echo "Wait for integration server to be Ready"
                    oc wait --for=condition=Ready integrationserver/${SERVER_NAME} --timeout=120s -n ${NAMESPACE}
                    '''
                }
            }
            stage('Unit Test') {
            environment {
                SERVER_NAME = "${serverName}"
                NAMESPACE = "${namespace}"
                OC_CREDS = credentials('oc-credentials')
            }
            agent {
                docker { image "${ocImage}"
                args '--entrypoint=""'
                reuseNode true
                }
            }
            steps {
                sh label: '', script: '''#!/bin/bash
                    export KUBECONFIG=$WORKSPACE/.kube/config
                    HOSTNAME=$(oc get route -n ${NAMESPACE} ${SERVER_NAME}-http -ogo-template --template='{{.spec.host}}')
                    curl -k http://${HOSTNAME}/api/v1/${SERVER_NAME} | jq -r .
                '''
            }
        }
    }
}