#!/usr/bin/env groovy
//=====================
// DEFINE ALL VARIABLES
//=====================
def projectName = "frontend"
def appPrefix = "frontend"
def containerRegistry = "demoabc.azurecr.io"

// Define deployment env from branch name
switch(env.BRANCH_NAME) {
    case "main":
        azureCredentialsId = "aks-prod"
        deployEnv ="production"
        aksClusterName = "k8s-prod"
        aksResourceGroupName = "demoabc"
        k8sNameSpaceID = "prod"
        crNameSpace = 'prod'
        replicaCount = "5"
        frontendHost = "app.prod.com"
        break
    case "develop":
        azureCredentialsId = "aks-test"
        deployEnv = "test"
        aksClusterName = "k8s-test"
        aksResourceGroupName = "demoabc"
        k8sNameSpaceID = "test"
        replicaCount = "3"
        crNameSpace = 'test'
        frontendHost = "app.test.dev"
        break
    default:
        azureCredentialsId = "aks-test"
        deployEnv = "test"
        aksClusterName = "k8s-test"
        aksResourceGroupName = "demoabc"
        k8sNameSpaceID = "${appPrefix}-test"
        replicaCount = "3"
        crNameSpace = 'test'
        frontendHost = "app.test.dev"
        break
}

//========================================================================
// Anything below this is universal and usually doesn't need to be changed
//========================================================================
echo "BUILD CAUSE: ${currentBuild.buildCauses}"

// Define job base name (without folders path in Jenkins)

def generateVersion() {
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'github-build-username-password', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD']]) {
        def zver = sh (script: "gitversion /showvariable PreReleaseLabel /url ${env.GIT_URL} /b ${env.BRANCH_NAME}  /u ${env.GIT_USERNAME} /p ${env.GIT_PASSWORD} /nocache", returnStdout: true)
        currentBuild.displayName = "${env.BUILD_ID}-${zver.toLowerCase().trim()}"
        return "${env.BUILD_ID}-${zver.toLowerCase().trim()}"
    }
}

pipeline {
    agent any
    
    options {
        disableConcurrentBuilds()
    }
    
    parameters {
        booleanParam(
            name: 'SKIP_BUILD',
            defaultValue: false,
            description: 'Run Control')
        booleanParam(
            name: 'RUN_UNIT_TESTS',
            defaultValue: false,
            description: 'Run Control')
        booleanParam(
            name: 'BUILD_RUNTIMES',
            defaultValue: false,
            description: 'Run Control')
        booleanParam(
            name: 'PUBLISH_RUNTIME',
            defaultValue: false,
            description: 'Run Control')
        booleanParam(
            name: 'DEPLOY_BRANCH',
            defaultValue: false,
            description: 'Run Control')
        booleanParam(
            name: 'RUN_REGRESSION_TESTS',
            defaultValue: false,
            description: 'Run Control')
        booleanParam(
            name: 'RUN_SMOKE_TESTS',
            defaultValue: false,
            description: 'Run Control')
    }

    environment {
        VERSION = "${(generateVersion())}"
        ngx_srv = "${ngx_srv}"
        frontendHost = "${frontendHost}"
        containerRegistry = "${containerRegistry}"
        crNameSpace = "${crNameSpace}"
        projectName = "${projectName}"
        buildID = "${env.BUILD_ID}"
        k8sNameSpaceID = "${k8sNameSpaceID}"
        replicaCount = "${replicaCount}"
        deployEnv = "${deployEnv}"
    }

    stages {

        stage('Update Configuration') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'main'
                    equals expected: true, actual: params.BUILD_RUNTIMES
                }
                beforeOptions true
                beforeInput true
                beforeAgent true
            }
            steps {
                echo 'Modifying Configuration File...'
                fileOperations([
                    fileDeleteOperation(
                        includes: '.env',
                        excludes: ''
                    ),
                    fileRenameOperation(
                        source: "_env.${deployEnv}",
                        destination: '.env'
                    )
               ])
            }
        }

        stage ('Yarn Install') {
            when {
                allOf {
                    equals expected: false, actual: params.SKIP_BUILD
                }
                beforeOptions true
                beforeInput true
                beforeAgent true
            }
            environment {
                JFROG_USER="build"
            }
            steps {
                script {
                    docker.withRegistry("https://${containerRegistry}", containerRegistry ) {
                        pkgImage = docker.build("${projectName}-pkg:${VERSION}", "-f ./.jenkins/Dockerfile.Yarn.Install ${WORKSPACE}") 
                    }
                }
            }    
        }

        stage('Build SDK Image(s)') {
            when {
                allOf {
                    equals expected: false, actual: params.SKIP_BUILD
                }
                beforeOptions true
                beforeInput true
                beforeAgent true
            }
            failFast false
            parallel {
                stage ('Yarn Build') {
                    steps {
                        script {
                            rxImage = docker.build("${containerRegistry}/${crNameSpace}/${projectName}-rx-sdk:${VERSION}", "--force-rm -f ${WORKSPACE}/.jenkins/Dockerfile.Yarn.Build -t ${projectName}-rx-sdk:${VERSION} --build-arg BASEIMG=${projectName}-pkg --build-arg IMG_VERSION=${VERSION} ${WORKSPACE}")
                        }
                    }
                }
            }
        }

        stage('Execute Test Suites') {
            when {
                allOf{
                    equals expected: false, actual: params.SKIP_BUILD
                    anyOf {
                        branch 'main'
                        branch 'develop'
                        branch 'PR*'
                        equals expected: true, actual: params.RUN_UNIT_TESTS
                        equals expected: true, actual: params.RUN_INT_TESTS
                    }                    
                }
                beforeOptions true
                beforeInput true
                beforeAgent true
            }
            failFast true
            parallel {
                stage ('Run React Unit Tests') {
                    when {
                        anyOf {
                            branch 'main'
                            branch 'develop'
                            branch 'PR*'
                            equals expected: true, actual: params.RUN_UNIT_TESTS
                        }
                        beforeOptions true
                        beforeInput true
                        beforeAgent true
                    }
                    steps {
                        script {
                            utrxImage = docker.build("${projectName}-ut:${VERSION}", "--force-rm --no-cache -f ${WORKSPACE}/.jenkins/Dockerfile.Yarn.Build.Test --build-arg BASEIMG=${projectName}-rx-sdk --build-arg IMG_VERSION=${VERSION} ${WORKSPACE}/.jenkins")
                        }
                    }
                }
            }
        }


        stage('Process Test Suites') {
            when {
                allOf{
                    equals expected: false, actual: params.SKIP_BUILD
                    anyOf {
                        branch 'main'
                        branch 'develop'
                        branch 'PR*'
                        equals expected: true, actual: params.RUN_UNIT_TESTS
                        equals expected: true, actual: params.RUN_INT_TESTS
                    }                    
                }
                beforeOptions true
                beforeInput true
                beforeAgent true
            }
            failFast true
            parallel {
                stage ('Process Unit Tests') {
                    when {
                        anyOf {
                            branch 'main'
                            branch 'develop'
                            branch 'PR*'
                            equals expected: true, actual: params.RUN_UNIT_TESTS
                        }
                        beforeOptions true
                        beforeInput true
                        beforeAgent true
                    }
                    steps {    
                        script {
                            utrxImage.inside {
                                pwsh "Copy-Item -Path /src/junit.xml -Destination ${WORKSPACE}/junit.xml -Verbose -Force"
                                pwsh "Copy-Item -Path /src/coverage/cobertura-coverage.xml -Destination ${WORKSPACE}/cobertura-coverage.xml -Verbose -Force"
                                xunit(
                                    testTimeMargin: '600000',
                                    thresholdMode: 1,
                                    thresholds: [failed(), skipped()],
                                    tools: [JUnit(deleteOutputFiles: true, failIfNotNew: true, pattern: "junit.xml", skipNoTestFiles: false, stopProcessingIfError: true)]
                                )
                                catchError {
                                    cobertura coberturaReportFile: "cobertura-coverage.xml"
                                }
                            }
                        }
                    }
                }
            }
        }


        stage('Build Runtime Image(s)') {
            when {
                allOf {
                    equals expected: false, actual: params.SKIP_BUILD
                    anyOf {
                        branch 'main'
                        branch 'develop'
                        equals expected: true, actual: params.BUILD_RUNTIMES
                        equals expected: true, actual: params.PUBLISH_RUNTIME
                    }
                }
                beforeOptions true
                beforeInput true
                beforeAgent true
            }
            failFast false
            parallel {
                stage ('Build React Runtim Image') {
                    steps {
                        script {
                            rtImageReact = docker.build("${containerRegistry}/${crNameSpace}/${projectName}-rx-runtime:${VERSION}", "--force-rm -f ${WORKSPACE}/.jenkins/Dockerfile.Runtime.React --build-arg BASEIMG=${projectName}-rx-sdk --build-arg IMG_VERSION=${VERSION} ${WORKSPACE}/.jenkins")
                        }
                    }
                }
            }
        }

        stage ('Push Image(s)') {
            when {
                allOf {
                    equals expected: false, actual: params.SKIP_BUILD
                    anyOf {
                        branch 'main'
                        branch 'develop'
                        allOf {
                            equals expected: true, actual: params.PUBLISH_RUNTIME
                            equals expected: true, actual: params.BUILD_RUNTIMES
                        }
                    }
                }
            }
            steps {
                script {
                    docker.withRegistry("https://${containerRegistry}", containerRegistry ) {
                        rxImage.push()
                        rtImageReact.push()
                    }
                }
            }
        }
        

        stage ('Deploy Application Manifest') {
            when {
                allOf {
                    equals expected: false, actual: params.SKIP_BUILD
                    anyOf {
                        branch 'develop'
                        branch 'main'
                        allOf {
                            equals expected: true, actual: params.DEPLOY_BRANCH
                            equals expected: true, actual: params.PUBLISH_RUNTIME
                            equals expected: true, actual: params.BUILD_RUNTIMES
                        }
                    }
                }
            }
            steps {
                dir(".jenkins") {
                    acsDeploy azureCredentialsId: "${azureCredentialsId}",
                            sshCredentialsId: "",
                            resourceGroupName: "${aksResourceGroupName}",
                            containerService: "${aksClusterName} | AKS",
                            configFilePaths: "*aks-manifest.*",
                            enableConfigSubstitution: true
                }
            }
        }
    }
}
