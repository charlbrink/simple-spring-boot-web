/**
 * Jenkins pipeline to build an application with the GitHub flow in mind (https://guides.github.com/introduction/flow/).
 *
 * This pipeline requires the following credentials:
 * ---
 * Type          | ID                | Description
 * Secret text   | devops-project    | The OpenShift project Id of the DevOps project that this Jenkins instance is running in
 * Secret text   | dev-project       | The OpenShift project Id of the project's development environment
 * Secret text   | sit-project       | The OpenShift project Id of the project's integration testing environment
 * Secret text   | uat-project       | The OpenShift project Id of the project's user acceptance environment
 *
 */

// TODO extract common stuff into shared libraries: https://jenkins.io/doc/book/pipeline/shared-libraries/

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

  def teamDevOpsProject
  def projectDevProject
  def projectSitProject
  def projectUatProject

  withCredentials([
          string(credentialsId: 'devops-project', variable: 'DEVOPS_PROJECT_ID'),
          string(credentialsId: 'dev-project', variable: 'DEV_PROJECT_ID'),
          string(credentialsId: 'sit-project', variable: 'SIT_PROJECT_ID'),
          string(credentialsId: 'uat-project', variable: 'UAT_PROJECT_ID'),
  ]) {
    teamDevOpsProject = "${env.DEVOPS_PROJECT_ID}"
    projectDevProject = "${env.DEV_PROJECT_ID}"
    projectSitProject = "${env.SIT_PROJECT_ID}"
    projectUatProject = "${env.UAT_PROJECT_ID}"
  }

  def project = "${env.JOB_NAME.split('/')[0]}"
  def app = "${env.JOB_NAME.split('/')[1]}"
  def appBuildConfig = "${project}-${app}"
  def tag

  stage('SCM Checkout') {

    final scmVars = checkout(scm)
    def shortGitCommit = scmVars.GIT_COMMIT[0..6]
    def pom = readMavenPom file: pomFileLocation

    tag = "${pom.version}-${shortGitCommit}"
    echo "Building application ${app}:${tag} from commit ${scmVars} with BuildConfig ${appBuildConfig}"
    sh "orig=\$(pwd); cd \$(dirname ${pomFileLocation}); git describe --tags; cd \$orig"
  }

  stage ('Checks and Build') {

    sh "${mvnCmd} clean install -DskipTests=true -f ${pomFileLocation}"

  }

  stage ('Unit Test') {

    sh "${mvnCmd} test -f ${pomFileLocation}"

  }

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
