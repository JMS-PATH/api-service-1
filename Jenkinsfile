def version
def appName = 'api-service-1'
def imgName = 'service-1'
def dockerHubImg = "aswinrprasad/${imgName}"
def dockerGitImg = "ghcr.io/jms-path/${appName}/${imgName}"
def branchName = env.BRANCH_NAME

pipeline {
    agent any

    tools {
        maven "M3"
    }
    options {
        buildDiscarder(logRotator(numToKeepStr: "10", artifactNumToKeepStr: "10"))
    }
    stages {
        stage("Initialize Pipeline"){
            steps {
                echo "Starting the Jenkins pipeline for eureka server"
                echo "Fetching app version.."
                script{
                    version = readMavenPom().getVersion()
                    mvnGoals = "install"
                    if( branchName == "main" ) {
                        version = version.replace("-SNAPSHOT", "")
                        mvnGoals = "clean versions:set -DremoveSnapshot deploy"
                    }
                    else if( branchName == "test" ) {
                        mvnGoals = "deploy"
                        version = version.replace("-SNAPSHOT", "")+"-test"
                    }
                    else if (branchName.startsWith("hotfix/") || branchName.startsWith("feature/")){
                       version = version.replace("-SNAPSHOT", "")+branchName.replaceFirst(/^(hotfix\/|feature\/)/, "")+"-fh"
                    }
                    else {
                        version = version+"-dev"
                    }
                }
            }
        }

        stage("Maven Build") {
            steps {
                echo "Version is ${version}"
                echo "Building Maven Project"
                sh "mvn clean compile"
            }
        }

        stage("Scan Open Source Libraries with Snyk") {
            steps {
                echo "Scanning all open source library jar files with snyk"
                sh "chmod +x mvnw"
                snykSecurity(
                  snykInstallation: 'snyk-scanner',
                  snykTokenId: 'snyk-credentials',
                  projectName: appName,
                  failOnIssues: false
                )
            }
        }

        stage("Sonar Code Quality Scan") {
            steps {
                echo "Scanning code quality with SonarQube"
                script {
                    withSonarQubeEnv(credentialsId: 'sonarqube-credentials', installationName: 'sonar-scanner') {
                        sh 'mvn sonar:sonar -Dsonar.projectName='+appName
                    }
                }
            }
        }

        stage("Deploy Artefact to Nexus Repo") {
            when {
                anyOf {
                   branch 'main'
                   branch 'test'
               }
            }
            steps {
                echo "Packaging and deploying artefact to NXRM"
                sh "mvn ${mvnGoals} -Dmaven.test.skip=true"
            }
        }

        stage("Deploy image to DockerHub and GitHub"){
            steps {

                sh "docker build -t ${dockerHubImg}:${version} ."
                sh "docker tag ${dockerHubImg}:${version} ${dockerGitImg}:${version}"
                script{
                    if (branchName == "main") {
                        // For DockerHub
                        sh "docker tag ${dockerHubImg}:${version} ${dockerHubImg}:stable"
                        withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', passwordVariable: 'pass', usernameVariable: 'uname')]) {
                            sh "docker login -u ${uname} -p ${pass}"
                            sh "docker push ${dockerHubImg}:stable"
                            sh "docker push ${dockerHubImg}:${version}"
                        }

                        // For GitHub
                        withCredentials([usernamePassword(credentialsId: 'github-credentials', passwordVariable: 'pass', usernameVariable: 'uname')]) {
                            sh "docker login -u ${uname} -p ${pass} ghcr.io"
                            sh "docker tag ${dockerGitImg}:${version} ${dockerGitImg}:stable"
                            sh "docker push ${dockerGitImg}:stable"
                            sh "docker push ${dockerGitImg}:${version}"
                        }
                    }
                    else if(branchName == "test"){
                        // For dockerHub
                        withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', passwordVariable: 'pass', usernameVariable: 'uname')]) {
                            sh "docker login -u ${uname} -p ${pass}"
                            sh "docker push ${dockerHubImg}:${version}"
                        }
                        // For GitHub
                        withCredentials([usernamePassword(credentialsId: 'github-credentials', passwordVariable: 'pass', usernameVariable: 'uname')]) {
                            sh "docker login -u ${uname} -p ${pass} ghcr.io"
                            sh "docker push ${dockerGitImg}:${version}"
                        }
                    }
                    else {
                        withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', passwordVariable: 'pass', usernameVariable: 'uname')]) {
                            sh "docker login -u ${uname} -p ${pass}"
                            sh "docker push ${dockerHubImg}:${version}"
                        }
                    }
                }
            }
        }

        stage("Docker Image Clean Up after Build") {
            steps {
                sh "docker image prune -af"
            }
        }
    }
}