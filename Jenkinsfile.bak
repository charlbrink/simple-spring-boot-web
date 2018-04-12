#!/usr/bin/groovy

////
// This pipeline requires the following plugins:
// Kubernetes Plugin 0.10
////


node('master') {
  
  stage('SCM Checkout') {
    
    println("PARAM: ${params.GIT_REPO}");

    checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: ${params.GIT_REP}]]])
    sh "ls -lrt"

  }
}
