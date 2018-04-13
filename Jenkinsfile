#!/usr/bin/groovy

////
// This pipeline requires the following plugins:
// Kubernetes Plugin 0.10
////

String ocpApiServer = env.OCP_API_SERVER ? "${env.OCP_API_SERVER}" : "https://openshift.default.svc.cluster.local"

node('master') {

  env.NAMESPACE = readFile('/var/run/secrets/kubernetes.io/serviceaccount/namespace').trim()
  env.TOKEN = readFile('/var/run/secrets/kubernetes.io/serviceaccount/token').trim()
  env.OC_CMD = "oc --request-timeout='0' --token=${env.TOKEN} --server=${ocpApiServer} --certificate-authority=/run/secrets/kubernetes.io/serviceaccount/ca.crt --namespace=${env.NAMESPACE}"

  env.APP_NAME = "${env.JOB_NAME}".replaceAll(/-?pipeline-?/, '').replaceAll(/-?${env.NAMESPACE}-?/, '')
  def projectBase = "${env.NAMESPACE}".replaceAll(/-build/, '')
  env.STAGE1 = "${projectBase}-dev"
  env.STAGE2 = "${projectBase}-sit"

}

node('maven') {
  def mvnCmd = "mvn"
  String pomFileLocation = env.BUILD_CONTEXT_DIR ? "${env.BUILD_CONTEXT_DIR}/pom.xml" : "pom.xml"

  stage('SCM Checkout') {

    checkout scm
    sh "orig=\$(pwd); cd \$(dirname ${pomFileLocation}); git describe --tags; cd \$orig"

  }

  stage ('Checks and Build') {

    sh "${mvnCmd} clean install -DskipTests=true -f ${pomFileLocation}"

  }

  stage ('Unit Test') {

    sh "${mvnCmd} test -f ${pomFileLocation}"

  }

  // The following variables need to be defined at the top level and not inside
  // the scope of a stage - otherwise they would not be accessible from other stages.
  // Extract version and other properties from the pom.xml
  def groupId    = getGroupIdFromPom("./pom.xml")
  def artifactId = getArtifactIdFromPom("./pom.xml")
  def version    = getVersionFromPom("./pom.xml")
  println("Artifact ID:" + artifactId + ", Group ID:" + groupId)
  println("New version tag:" + version)

  if (env.BRANCH_NAME == 'master' || !env.BRANCH_NAME) {
    stage('OpenShift Build Image') {
      sh """
                rm -rf oc-build && mkdir -p oc-build/deployments

                for t in \$(echo "jar;war;ear" | tr ";" "\\n"); do
                    cp -rfv ./target/*.\$t oc-build/deployments/ 2> /dev/null || echo "No \$t files"
                done

                ${env.OC_CMD} start-build ${env.APP_NAME} --from-dir=oc-build --wait=true --follow=true || exit 1
            """
    }

    stage("Deploy to ${env.STAGE1}") {
      sh ": Deploying to ${env.STAGE1}..."

      sh """
                ${env.OC_CMD} tag ${env.NAMESPACE}/${env.APP_NAME}:${tag} ${env.STAGE1}/${env.APP_NAME}:${tag}
            """

    }

    stage("Verify Deployment to ${env.STAGE1}") {

      openshiftVerifyDeployment(deploymentConfig: "${env.APP_NAME}", namespace: "${STAGE1}", verifyReplicaCount: true)

    }

    stage('Smoke Test') {

      //TODO: Add application integration testing, verify db connectivity, rest calls ...
      //sh "${mvnCmd} -Psmoke test -f ${pomFileLocation}"
      //sh "${mvnCmd} -Dtest=*/smoke/*/*Test.java test -f ${pomFileLocation}"

    }

    stage('Integration Test') {

      //TODO: Add application integration testing, verify db connectivity, rest calls ...
      //sh "${mvnCmd} -Pe2e test -f ${pomFileLocation}"
      //sh "${mvnCmd} -Dtest=*/e2e/*/*Test.java test -f ${pomFileLocation}"

    }

    stage("Deploy to ${env.STAGE2}") {
      sh ": Deploying to ${env.STAGE2}..."

      sh """
                ${env.OC_CMD} tag ${env.NAMESPACE}/${env.APP_NAME}:${tag} ${env.STAGE2}/${env.APP_NAME}:${tag}
            """

    }

    stage("Verify Deployment to ${env.STAGE2}") {

      openshiftVerifyDeployment(deploymentConfig: "${env.APP_NAME}", namespace: "${STAGE2}", verifyReplicaCount: true)

    }

  }
}

// Convenience Functions to read variables from the pom.xml
// Do not change anything below this line.
def getVersionFromPom(pom) {
  def matcher = readFile(pom) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}
def getGroupIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<groupId>(.+)</groupId>'
  matcher ? matcher[0][1] : null
}
def getArtifactIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<artifactId>(.+)</artifactId>'
  matcher ? matcher[0][1] : null
}
