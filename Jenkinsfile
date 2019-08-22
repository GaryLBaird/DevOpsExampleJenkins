#!groovy
pipeline {
    agent { node { label 'jenkins-devops' } }
    parameters {
        // git parameters are used in the Jenkins pipeline configuration
        gitParameter(name: 'SOURCE_HASH', type: 'PT_REVISION', description: 'What branch hash is the Pull Request coming from?')
        gitParameter(name: 'SOURCE_BRANCH', type: 'PT_BRANCH', description: 'What branch is the Pull Request coming from?', branchFilter: 'origin/(.*)', sortMode: 'ASCENDING_SMART')
        gitParameter(name: 'TARGET_BRANCH', type: 'PT_BRANCH', description: 'What branch is the Pull Request targeting?', branchFilter: 'origin/(.*)', sortMode: 'DESCENDING_SMART', defaultValue: 'develop')
        // these parameters are strictly for notifications of the Pull Request build outcome if a failure occurs, they are not required
        string(name: 'NOTIFY_EMAILS', description: 'What email(s) should we use to notify when issues with the Pull Request occur?', defaultValue: '')
        string(name: 'PULL_REQUEST_URL', description: 'What is the URL of the Pull Request?', defaultValue: '')
    }
    environment {
        NODEJS_HOME = "${tool 'Node-v10.15.1-win-x64'}"
        PATH="e:\\tools\\git\\bin\\;e:\\tools\\;${env.PATH};${env.NODEJS_HOME}"
    }
    stages {
        stage("Config") {
            steps {
                step([$class: "StashNotifier"])
                script {
                    powershell(script: "./build/src/CreateConfigForBuild.ps1")
                }
            }
        }

        stage("Client Build") {
            steps {
                script {
                    env.CATSCI=true
                    dir("DotNetAppName/build/src") {
                        powershell(script: "./BuildClient.ps1 pr")
                    }
                }
            }
        }

        stage("Server Build") {
            steps {
                script {
                    dir("DotNetAppName/build/src") {
                        powershell(script: "./BuildServer.ps1 -Config Release")
                        powershell(script: "./PublishServer.ps1 -Config Release")
                    }
                }
            }
        }

        stage("Client Tests") {
            steps {
                script {
                    //env.DotNetAppNameCI=true
                    dir("DotNetAppName/client/cats") {
                        try {
                            bat "npm run test-ci"
                        }
                        catch (Exception ex) {
                            currentBuild.result = "UNSTABLE"
                        }
                        finally {
                            junit "**/test-results.xml"
                            publishHTML([allowMissing: true,
                                        alwaysLinkToLastBuild: true,
                                        keepAll: true,
                                        reportDir: "coverage",
                                        reportFiles: "DotNetAppName-ui//index.html, app-layout//index.html, DotNetAppName-layout//index.html, dashboard//index.html, bop//index.html",
                                        reportName: "Code Coverage",
                                        reportTitles: "DotNetAppName-ui, app-layout, DotNetAppName-layout, dashboard, bop"])
                            bat "npm run lint"
                        }
                    }
                }
            }
        }

        stage("Server Tests") {
            steps {
                script {
                    try {
                        powershell(script: "./DotNetAppName/build/src/RunServerTests.ps1 -Config Release")
                    }
                    catch (Exception ex) {
                        currentBuild.result = "UNSTABLE"
                    }
                    finally {
                        echo 'Loading Results'
                        step([$class: 'MSTestPublisher', testResultsFile:"**/build/test-output/server/*.trx", failOnError: true, keepLongStdio: true])                                            }
                }
            }
        }
    }
    post {
        always {
            script {
                currentBuild.result = currentBuild.result ?: "SUCCESS"
                step([$class: "StashNotifier"])
            }
        }
        unstable {
            script {
                if (params.NOTIFY_EMAILS) {
                    try {
                        mail to: params.NOTIFY_EMAILS,
                        subject: "[Jenkins] ${currentBuild.fullDisplayName} has failing tests",
                        body: "The unit tests are failing for your Pull Request build in Jenkins.\nBuild: ${env.BUILD_URL}\n\nPull Request: ${params.PULL_REQUEST_URL}\n\nSource: ${params.SOURCE_BRANCH} ('${params.SOURCE_HASH}')\nTarget: ${params.TARGET_BRANCH}"
                    }
                    catch (Exception ex) {
                        echo "Unable to send unstable email to '${params.NOTIFY_EMAILS}'."
                    }
                }
                else {
                    echo "Unable to send unstable email. NOTIFY_EMAILS not specified."
                }
            }
        }
        failure {
            script {
                if (params.NOTIFY_EMAILS) {
                    try {
                        mail to: params.NOTIFY_EMAILS,
                        subject: "[Jenkins] ${currentBuild.fullDisplayName} failed",
                        body: "Something is wrong with your Pull Request build in Jenkins.\nBuild: ${env.BUILD_URL}\n\nPull Request: ${params.PULL_REQUEST_URL}\n\nSource: ${params.SOURCE_BRANCH} ('${params.SOURCE_HASH}')\nTarget: ${params.TARGET_BRANCH}"
                    }
                    catch (Exception ex) {
                        echo "Unable to send failure email to '${params.NOTIFY_EMAILS}'."
                    }
                }
                else {
                    echo "Unable to send failure email. NOTIFY_EMAILS not specified."
                }
            }
        }
    }
}
