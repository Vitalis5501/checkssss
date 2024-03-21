import groovy.transform.Field
@Library(['fidelity-pipeline-library@release/0'])

//@Field def Email = "MDSQUADHISTORYMAKERS@fmr.com"
@Field def E_Body = ""
@Field def GIT_REPO_URL = ""
@Field def GIT_REPO_BRANCH = ""
@Field def PATH_TO_POM = ""
@Field def JAVA_VERSION = ""
@Field def MAVEN_GOAL = ""
@Field def REPORT_FILE_LOCATION = ""

def lib_branch = env.BRANCH_NAME
library "pipeline-platformengineering-shared-libraries@${lib_branch}"

pipeline {
    agent {
        kubernetes{
            defaultContainer "ubn22-aws-utils"
            yaml buildpack(
                pack(name: "ubn22-mvn3-java8-nodejs18", version: "stable"),
                pack(name: "ubn22-mvn3-java11-nodejs18", version: "stable"),
                pack(name: "ubn22-mvn3-java17-nodejs18", version: "stable"),
                pack(name: "ubn22-aws-utils", version: "stable")
            )
        }
    }

  options { timestamps () }

    environment {
        VAULT_MAPPINGS = vaultJCV2Mapping(
            SRV_ACCT_USR: 'pl000102/ffiopleng/pipe_user',
            SRV_ACCT_PSW: 'pl000102/ffiopleng/pipe_password',
            PIPELINE_USR: 'pl000102/ffiopleng/pipe_user',
            PIPELINE_PSW: 'pl000102/ffiopleng/pipe_password',
            AWS_USR: 'pl000102/ffiopleng/pipe_user',
            AWS_PSW: 'pl000102/ffiopleng/pipe_password',
            OAUTH_USR: 'pl000102/ffiopleng/pipe_user',
            OAUTH_PSW: 'pl000102/ffiopleng/pipe_password',
            GITHUB_TOKEN: 'pl000102/ap126999/github/github_token',
            JIRA_KEY: 'pl000102/ap126999/jira_api/api_key',
            JIRA_SEC: 'pl000102/ap126999/jira_api/api_sec',
            JIRA_CRED: 'pl000102/ap126999/jira_api/api_cred'
        )
        AWS_ACCESS_KEY_ID = ""
        AWS_SECRET_ACCESS_KEY = ""
        def AWS_SESSION_TOKEN = ""
        PIPE_ENV = "dev"
    }

    parameters {
        string(name: 'GIT_REPO_URL', defaultValue: "none", description: 'test repo')
        string(name: 'GIT_REPO_BRANCH', defaultValue: "none", description: 'test branch')
        string(name: 'PATH_TO_POM', defaultValue: ".", description: 'path to pom')
        string(name: 'JAVA_VERSION', defaultValue: 'java8', description: 'java8|java11|java17')
        string(name: 'MAVEN_GOAL', defaultValue: '', description: 'maven commands. eg - clean install')
        string(name: 'REPORT_FILE_LOCATION', defaultValue: '', description: 'maven report file location. eg - target/cucumber-report/CucumberTestReport.json')
        string(name: 'Email', defaultValue: 'MDSQUADHISTORYMAKERS@fmr.com', description: 'Email address for notifications')
        string(name: 'AWS_ACCOUNT', defaultValue: '', description: 'AWS account number')
        string(name: 'AWS_ROLE', defaultValue: '', description: 'AWS role')


    }

    stages {
        stage ('Initial Setup') {
            steps{
                script{
                    //sending pre email
                    def start_email = "Starting Smoke Test Automation: <br><br>You may track it here: ${RUN_DISPLAY_URL} <br><br> or here: ${BUILD_URL}"
                    def start_subject = "SMOKE TEST - Automation Test Triggered"
                    emailext body: start_email, recipientProviders: [requestor()], subject: start_subject, to: params.Email
                }
            }
        }
        stage ('AWS Authentication') {
            steps {
                script {
                    awsAuth(
                        account: params.AWS_ACCOUNT,
                        role: params.AWS_ROLE,
                        domain: 'dmn1',
                        profile: "default",
                        additionalArgs: ""
                    )
                    sh '''
                        #!/bin/bash
                        AWS_ACCESS_KEY_ID_JENKINS=\$(sed -n 's/aws_access_key_id = \\(.*\\)/\\1/p' ~/.aws/credentials)
                        AWS_SECRET_ACCESS_KEY_JENKINS=\$(sed -n 's/aws_secret_access_key = \\(.*\\)/\\1/p' ~/.aws/credentials)
                        AWS_SESSION_TOKEN_JENKINS=\$(sed -n 's/aws_session_token = \\(.*\\)/\\1/p' ~/.aws/credentials)
                        echo $AWS_ACCESS_KEY_ID_JENKINS
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID_JENKINS
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY_JENKINS
                        export AWS_SESSION_TOKEN=$AWS_SESSION_TOKEN_JENKINS
                        USER root
                        apt-get update 
                        apt-get install -y openjdk-11-jdk
                        USER 1001
                    '''
                    
                }
            }
        }
        stage ('Checkout') {
            steps{
                withPipelineCredentials(){
                    script{
                        //setup java version
                        env.java_container = "ubn22-mvn3-${params.JAVA_VERSION}-nodejs18"
                        echo "java container: ${java_container}"
                        
                        // Install Java 11 SDK
                        
                        
                        
                        env.GIT_USR = PIPELINE_USR
                        env.GITHUB_USR = PIPELINE_USR
                        env.GIT_PSW = PIPELINE_PSW

                        //clone repo
                        gitScmClone(
                            repository: params.GIT_REPO_URL,
                            branch: params.GIT_REPO_BRANCH,
                            targetDir: "."                
                        )
                    }
                }
            }
        }

        stage("Maven Test") {
            steps {
                container(java_container){
                    script {
                        echo "starting maven test"
                        sh echo $AWS_ACCESS_KEY_ID
                        mvn(goal: params.MAVEN_GOAL, pom: params.PATH_TO_POM)
                    }
                }
            }
        }

        stage("Push Report to JIRA") {
            steps {
                script {
                    withPipelineCredentials() {
                    boolean post_result = false
                    def msg = ""
                    (post_result, msg) = jira.pushReportToJira(params.REPORT_FILE_LOCATION)
                    if(( post_result )){
                        E_Body = E_Body + "<br><br> Token generation SUCCESSFUL"
                    }
                    else{
                        E_Body = E_Body + "<br><br>ERROR Token generation was not successful <br><br>${msg}"
                        error "ERROR Token generation was not successful "
                    }
                    }
                }
            }
        }      
    }

  post { 
    unsuccessful { 
        script{
            if (( !(params.Email) )){
                echo "email is empty"
                //Email = "MDSQUADHISTORYMAKERS@fmr.com"
            }
            echo "sending email to this address ${params.Email}"
            def E_subject = "SMOKE TEST - Automation Test FAILED"
            E_Body = "Build FAILED: <br><br>" + E_Body + " <br><br>Check logs here for more info: ${BUILD_URL}"
            emailext body: E_Body, recipientProviders: [requestor()], subject: E_subject, to: params.Email
        }
    }
	success { 
        script{
            if (( !(params.Email) )){
                echo "email is empty"
                //Email = "MDSQUADHISTORYMAKERS@fmr.com"
            }
            echo "sending email to this address ${params.Email}"
            def E_subject = "SMOKE TEST - Automation Test SUCCESS"
            E_Body = "SMOKE TEST SUCCESSFUL: <br><br>" + E_Body + "<br><br>Check logs here for more info: ${BUILD_URL}"
            emailext body: E_Body, recipientProviders: [requestor()], subject: E_subject, to: params.Email
        }
    }
    aborted {
        script{
            if (( !(params.Email) )){
                echo "email is empty"
                //Email = "MDSQUADHISTORYMAKERS@fmr.com"
            }
            echo "sending email to this address ${params.Email}"
            def E_subject = "SMOKE TEST - Automation Test ABORTED"
            E_Body = "Build ABORTED Unexpectantly: <br><br>" + E_Body + " <br><br>Check logs here for more info: ${BUILD_URL}"
            emailext body: E_Body, recipientProviders: [requestor()], subject: E_subject, to: params.Email
        }
    }
  }
}
