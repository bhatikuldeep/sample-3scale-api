#!groovy

library identifier: '3scale-toolbox-jenkins@master',
        retriever: modernSCM([$class: 'GitSCMSource',
                              remote: 'https://github.com/rh-integration/3scale-toolbox-jenkins.git',
                              traits: [[$class: 'jenkins.plugins.git.traits.BranchDiscoveryTrait']]])

def service = null

node() {
  stage('Checkout Source') {
    checkout scm
  }

  stage("Prepare") {
    service = toolbox.prepareThreescaleService(
        openapi: [filename: "specs/swagger.yaml"],
        environment: [ baseSystemName: "api",
                       privateBaseUrl: params.PRIVATE_BASE_URL ],
        toolbox: [ openshiftProject: params.NAMESPACE,
                   destination: params.TARGET_INSTANCE,
                   image: "quay.io/redhat/3scale-toolbox:master", // TODO: remove me once the final image is released
                   secretName: params.SECRET_NAME,
                   verbose: true],
        service: [:],
        applications: [
            [ name: "my-test-app", description: "This is used for tests", plan: "test", account: params.DEVELOPER_ACCOUNT_ID ]
        ],
        applicationPlans: [
          [ systemName: "test", name: "Test", defaultPlan: true, published: true ],
          [ systemName: "silver", name: "Silver" ],
          [ systemName: "gold", name: "Gold" ],
        ]
    )

    //echo "toolbox version = " + service.toolbox.getToolboxVersion()
  }

  stage("Import OpenAPI") {
    service.importOpenAPI()
    echo "Service with system_name ${service.environment.targetSystemName} created !"
  }

  stage("Create an Application Plan") {
    service.applyApplicationPlans()
  }

  stage("Create an Application") {
    service.applyApplication()
  }

  stage("Run integration tests") {
    // To run the integration tests when using APIcast SaaS instances, we need
    // to fetch the proxy definition to extract the staging public url
    def proxy = service.readProxy("sandbox")
    def userkey = service.applications[0].userkey
    sh """set -e
    echo "Public Staging Base URL is ${proxy.sandbox_endpoint}"
    echo "userkey is ${userkey}"
    curl -sfk -w "IP: %{http_code}\n" -o /dev/null ${proxy.sandbox_endpoint}/ip -H 'api-key: ${userkey}'    
    """
  }

  stage("Promote to production") {
    service.promoteToProduction()
  }

}