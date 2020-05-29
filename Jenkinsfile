#!groovy

/*
 * This pipeline will provision the Beer Catalog API found on the microcks/api-lifecycle
 * repository (secured using API Key) on a 3scale Hosted instance. 
 * 
 * Setup instructions:
 * 1. Spin up a Jenkins master in a fresh OpenShift project
 * 2. Create a secret containing your 3scale_toolbox remotes:
 *
 *    3scale remote add 3scale-saas "https://123...456@MY-TENANT-admin.3scale.net/"
 *    oc create secret generic 3scale-toolbox --from-file="$HOME/.3scalerc.yaml"
 *
 * 3. If you chosed a different remote name, please adjust the targetInstance variable
 *    accordingly.
 * 4. Create a new Jenkins pipeline using the content of this file.
 */

def targetSystemName = "api"
def targetInstance = "3scale-saas"
def privateBaseURL = "http://loyalty-customer-experience-api:8080"
def testUserKey = "azerty1234567890"
def developerAccountId = "2445582796064"

/*
 * Only needed when using self-managed APIcast or on-premises installation of 3scale
 */
def publicStagingBaseURL = "http://apicast-3scalegateway.192.168.64.12.nip.io:80" // change to something such as "http://my-staging-api.example.test" for self-managed APIcast or on-premises installation of 3scale
def publicProductionBaseURL = "http://apicast-3scalegateway.192.168.64.12.nip.io:80" // change to something such as "http://my-production-api.example.test" for self-managed APIcast or on-premises installation of 3scale

node() {

  stage('Checkout Source') {
    checkout scm
  }

  stage("Fetch OpenAPI") {
    // Fetch the OpenAPI Specification file and provision it as a ConfigMap
    sh """
    oc delete configmap apirootfolder --ignore-not-found
    oc create configmap apirootfolder --from-file="=$(pwd)"
    """
  }

  stage("Import OpenAPI") {
    //def tooboxArgs = [ "3scale", "import", "openapi", "-d", targetInstance, "/specs/loyalty-customer-experience-api.yml", "--override-private-base-url=${privateBaseURL}", "-t", targetSystemName ]
    //3scale import openapi -d 3scale-saas -t api --override-private-base-url=http://loyalty-customer-experience-api:8080 --default-credentials-userkey=user_key --override-public-basepath=/consumer loyalty-customer-experience-api.yml 
    def tooboxArgs = [ "3scale", "import", "openapi", "-d", targetInstance, "-t", "api", "--override-private-base-url=${privateBaseURL}", "--default-credentials-userkey=${testUserKey}", "--override-public-basepath=/consumer", "/apirootfolder/specs/loyalty-customer-experience-api.yml"]
    if (publicStagingBaseURL != null) {
        tooboxArgs += "--staging-public-base-url=${publicStagingBaseURL}"
    }
    if (publicProductionBaseURL != null) {
        tooboxArgs += "--production-public-base-url=${publicProductionBaseURL}"
    }
    runToolbox(tooboxArgs)
  }
  
  stage("Create an Application Plan") {
    //runToolbox([ "3scale", "application-plan", "apply", targetInstance, targetSystemName, "test", "-n", "Test Plan", "--default" ])
    runToolbox([ "3scale", "application-plan", "import", targetInstance, targetSystemName, "--file=/apirootfolder/config/application-plan.yaml"])
    //3scale application-plan import icemobile-dev api --file=config/application-plan.yaml
  }

  stage("Create an Application") {
    runToolbox([ "3scale", "application", "apply", targetInstance, testUserKey, "--account=${developerAccountId}", "--name=Test Application", "--description=Created by Jenkins", "--plan=test", "--service=${targetSystemName}" ])
  }

  stage("Run integration tests") {
    /*
     * When using 3scale Hosted with hosted APIcast instance, we need to extract the proxy definition
     * to read the Public Staging Base URL. Otherwise, we can just re-use the publicStagingBaseURL
     * variable defined above.
     */
    if (publicStagingBaseURL == null) {
      def proxyDefinition = runToolbox([ "3scale", "proxy", "show", targetInstance, targetSystemName, "sandbox" ])
      def proxy = readJSON text: proxyDefinition
      publicStagingBaseURL = proxy.content.proxy.sandbox_endpoint
    }

    sh """
    echo "Public Staging Base URL is ${publicStagingBaseURL}"
    echo "userkey is ${testUserKey}"
    curl -vfk ${publicStagingBaseURL}/ip -H 'api-key: ${testUserKey}'
    curl -vfk ${publicStagingBaseURL}/json -H 'api-key: ${testUserKey}'
    """
  }
  
  stage("Promote to production") {
    runToolbox([ "3scale", "proxy", "promote", targetInstance, targetSystemName ])
  }

  stage("Publish ActiveDocs") {
    runToolbox([ "3scale", "activedocs", "apply", targetInstance, , "-p" ])
  }
}

/*
 * This function runs the 3scale toolbox as a Kubernetes Job.
 */
def runToolbox(args) {
  // You can adjust the Job Template to your needs
  def kubernetesJob = [
    "apiVersion": "batch/v1",
    "kind": "Job",
    "metadata": [
      "name": "toolbox"
    ],
    "spec": [
      // When the pipeline development is finished, you can increase the backoffLimit to 2
      "backoffLimit": 0,
      // Adjust the activeDeadlineSeconds according to your server velocity
      "activeDeadlineSeconds": 300,
      "template": [
        "spec": [
          "restartPolicy": "Never",
          "containers": [
            [
              "name": "job",
              // Do not forget to change the image tag to something more stable than "master"!
              "image": "quay.io/redhat/3scale-toolbox:master",
              "imagePullPolicy": "Always",
              "args": [ "3scale", "version" ],
              "env": [
                // This is needed for the 3scale_toolbox to read its configuration file
                // mounted from the toolbox-config secret 
                [ "name": "HOME", "value": "/config" ]
              ],
              "volumeMounts": [
                 [ "mountPath": "/config", "name": "toolbox-config" ],
                 [ "mountPath": "/apirootfolder", "name": "apirootfolder" ]  
              ]
            ]
          ],
          "volumes": [
            // This Secret contains the .3scalerc.yaml toolbox configuration file
            [ "name": "toolbox-config", "secret": [ "secretName": "3scale-toolbox" ] ],
            // This ConfigMap contains the artifacts to deploy (OpenAPI Specification file, Application Plan file, etc.)
            [ "name": "apirootfolder", "configMap": [ "name": "apirootfolder" ] ]            
          ]
        ]
      ]
    ]
  ]
  
  // Patch the Kubernetes job template to add the provided 3scale_toolbox arguments
  kubernetesJob.spec.template.spec.containers[0].args = args

  // Write the Kubernetes Job definition to a YAML file
  sh "rm -f -- job.yaml"
  writeYaml file: "job.yaml", data: kubernetesJob

  // Do some cleanup, create the job and wait a little bit...
  sh """
  oc delete job toolbox --ignore-not-found
  sleep 2
  oc create -f job.yaml
  sleep 20 # Adjust the sleep duration to your server velocity
  """
  
  // ...before collecting logs!
  def logs = sh(script: "oc logs -f job/toolbox", returnStdout: true)
  
  // When using "returnStdout: true", Jenkins does not display stdout logs anymore.
  // So, we have to display them by ourselves!
  echo logs

  // The stdout logs may contains parseable output, so we return them to the caller
  // that will use them as desired.
  return logs
}