#!/usr/bin/groovy

////
// This pipeline requires the following plugins:
// Kubernetes Plugin 0.10
////

String ocpApiServer = env.OCP_API_SERVER ? "${env.OCP_API_SERVER}" : "https://openshift.default.svc.cluster.local"

node('master') {

  env.NAMESPACE = readFile('/var/run/secrets/kubernetes.io/serviceaccount/namespace').trim()
  env.TOKEN = readFile('/var/run/secrets/kubernetes.io/serviceaccount/token').trim()
  env.OC_CMD = "oc --token=${env.TOKEN} --server=${ocpApiServer} --certificate-authority=/run/secrets/kubernetes.io/serviceaccount/ca.crt --namespace=${env.NAMESPACE}"

  env.APP_NAME = "${env.JOB_NAME}".replaceAll(/-?pipeline-?/, '').replaceAll(/-?${env.NAMESPACE}-?/, '')
  def projectBase = "${env.NAMESPACE}".replaceAll(/-dev/, '')
  env.STAGE1 = "${projectBase}-dev"
  env.STAGE2 = "${projectBase}-stage"
  env.STAGE3 = "${projectBase}-prod"

}

node('jenkins-slave-mvn') {
//  def mvnHome = "/usr/share/maven/"
//  def mvnCmd = "${mvnHome}bin/mvn"
  def mvnCmd = 'mvn'
  String pomFileLocation = env.BUILD_CONTEXT_DIR ? "${env.BUILD_CONTEXT_DIR}/pom.xml" : "pom.xml"

  stage('SCM Checkout') {
    checkout scm
  }

  stage('Build') {

    sh "${mvnCmd} clean install -DskipTests=true -f ${pomFileLocation}"

  }

  stage('Unit Test') {

     sh "${mvnCmd} test -f ${pomFileLocation} -Dmaven.test.skip=true"

  }

  // The following variables need to be defined at the top level and not inside
  // the scope of a stage - otherwise they would not be accessible from other stages.
  // Extract version and other properties from the pom.xml
  //def groupId    = getGroupIdFromPom("./pom.xml")
  //def artifactId = getArtifactIdFromPom("./pom.xml")
  //def version    = getVersionFromPom("./pom.xml")
  //println("Artifact ID:" + artifactId + ", Group ID:" + groupId)
  //println("New version tag:" + version)

  stage('Build Image') {

    sh """
      rm -rf oc-build && mkdir -p oc-build/deployments
      for t in \$(echo "jar;war;ear" | tr ";" "\\n"); do
        cp -rfv ./target/*.\$t oc-build/deployments/ 2> /dev/null || echo "No \$t files"
      done
      ${env.OC_CMD} start-build ${env.APP_NAME} --from-dir=oc-build --wait=true --follow=true || exit 1
    """
  }

  stage("Verify Deployment to ${env.STAGE1}") {

    openshiftVerifyDeployment(deploymentConfig: "${env.APP_NAME}", namespace: "${STAGE1}", verifyReplicaCount: true)

    input "Promote Application to Stage?"
  }

  stage("Promote To ${env.STAGE2}") {
    sh """
    ${env.OC_CMD} tag ${env.STAGE1}/${env.APP_NAME}:latest ${env.STAGE2}/${env.APP_NAME}:latest
    """
  }

  stage("Verify Deployment to ${env.STAGE2}") {

    openshiftVerifyDeployment(deploymentConfig: "${env.APP_NAME}", namespace: "${STAGE2}", verifyReplicaCount: true)

    input "Promote Application to Prod?"
  }

  stage("Promote To ${env.STAGE3}") {
    sh """
    ${env.OC_CMD} tag ${env.STAGE2}/${env.APP_NAME}:latest ${env.STAGE3}/${env.APP_NAME}:latest
    """
  }

  stage("Verify Deployment to ${env.STAGE3}") {

    openshiftVerifyDeployment(deploymentConfig: "${env.APP_NAME}", namespace: "${STAGE3}", verifyReplicaCount: true)

  }
}

println "Application ${env.APP_NAME} is now in Production!"
