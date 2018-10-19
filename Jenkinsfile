#!/usr/bin/env groovy
import groovy.json.JsonOutput

def gitRepo = params.GIT_REPO
def gitBranch = params.GIT_BRANCH != null && params.GIT_BRANCH != "" ? params.GIT_BRANCH : "master"

node('maven') {


 String env.threescaleurl = "https://ah-3scale-ansible-admin.app.rhdp.ocp.cloud.lab.eng.bos.redhat.com";    
 String env.apiaccesstoken = "845927b93be20fa491bf5601cc5e7fafa11d9d7eea8d70e7e46a79d35eab0aa2";    
 String env.appplanid = "17";
 String env.metricsid = "10";
 String env.ablimit = 25;
  
  
  env.uatnamespace = "fisdemo";
  env.prodnamespace = "fisdemoprod";


  // Get Source Code from SCM (Git) as configured in the Jenkins Project
  //stage('Checkout Source') {
    // For Jenkinsfile from GIT
    //checkout scm
    // for inline scripts
    //git url: gitRepo, branch: gitBranch
  //}

  def towerExtraVars = [
      git_repo: gitRepo,
      git_branch: gitBranch,
      threescale_cicd_api_backend_hostname: params.OPENSHIFT_SERVICE_NAME + ":8080",
      threescale_cicd_openapi_smoketest_operation: "GetLocation",
      threescale_cicd_api_backend_scheme: "http",
      threescale_cicd_api_base_system_name: "rhte_api",
      threescale_cicd_validate_openapi: false,
      openapi_file: "openapi-spec.yaml"
  ]

  //def thisPackage = readJSON file: 'package.json'
  //def currentVersion = thisPackage.version
  //def newVersion = "$currentVersion-$BUILD_NUMBER"

 

stage('moveToProd'){
      openshiftTag alias: "false",  destStream: "fisgateway-service", destTag: "latest", destinationNamespace: "fisdemoprod", namespace: "fisdemo", srcStream: "fisgateway-service-uat", srcTag: "uatready", verbose: "true"
  }


stage('StartNewServices') {
    print 'Start new service with one pod running' 
    openshiftScale depCfg: "fisgateway-service-new", namespace: "fisdemoprod", replicaCount: "1", verifyReplicaCount: "true", verbose: "true"
  }



  stage('UpdateRouteToAB') { 
    print 'deleteroute' 
    openshiftDeleteResourceByKey keys: "fisgateway-service", namespace: "fisdemoprod", types: "route", verbose: "true"
    
    print 'Update Route to only point to both new and stable service' 
    openshiftCreateResource jsonyaml: "{    'apiVersion': 'v1',    'kind': 'Route',    'metadata': {        'labels': {            'component': 'fisgateway-service-stable',            'group': 'quickstarts',            'project': 'fisgateway-service-stable',            'provider': 's2i',            'template': 'fisgateway-service',            'version': '1.0.0'        },        'name': 'fisgateway-service',        'namespace': 'fisdemoprod'    },    'spec': {        'alternateBackends': [            {                'kind': 'Service',                'name': 'fisgateway-service-new',                'weight': 30            }        ],        'host': 'fisgateway-service-fisdemoprod.master.rhdp.ocp.cloud.lab.eng.bos.redhat.com',        'to': {            'kind': 'Service',            'name': 'fisgateway-service-stable',            'weight': 70        },        'wildcardPolicy': 'None'    }}", namespace: "fisdemoprod", verbose: "false"
  }




  // Build the OpenShift Image in OpenShift using the artifacts from NPM
  // Also tag the image
  //stage('Build OpenShift Image') {
    // Trigger an OpenShift build in the dev environment
   // openshiftBuild bldCfg: params.OPENSHIFT_BUILD_CONFIG, checkForTriggeredDeployments: 'false',
                   //namespace: params.OPENSHIFT_BUILD_PROJECT, showBuildLogs: 'true',
                   //verbose: 'false', waitTime: '', waitUnit: 'sec', env: [ ]


    // Tag the new build
    //openshiftTag alias: 'false', destStream: params.OPENSHIFT_IMAGE_STREAM, destTag: "${newVersion}",
                 //destinationNamespace: params.OPENSHIFT_BUILD_PROJECT, namespace: params.OPENSHIFT_BUILD_PROJECT,
                // srcStream: params.OPENSHIFT_IMAGE_STREAM, srcTag: 'latest', verbose: 'false'

  //}

 // stage('Deploy API to test') {
    // Tag the new build as "ready-for-test"
    //openshiftTag alias: 'false', destStream: params.OPENSHIFT_IMAGE_STREAM, srcTag: "${newVersion}",
                 //destinationNamespace: params.OPENSHIFT_TEST_ENVIRONMENT, namespace: params.OPENSHIFT_BUILD_PROJECT,
                 //srcStream: params.OPENSHIFT_IMAGE_STREAM, destTag: 'ready-for-test', verbose: 'false'

    // Trigger a new deployment
    //openshiftDeploy deploymentConfig: params.OPENSHIFT_DEPLOYMENT_CONFIG, namespace: params.OPENSHIFT_TEST_ENVIRONMENT

    // Deploy the API to 3scale
    ansibleTower towerServer: params.ANSIBLE_TOWER_SERVER,
                 inventory: params.ANSIBLE_TEST_INVENTORY,
                 jobTemplate: params.ANSIBLE_JOB_TEMPLATE,
                 extraVars: JsonOutput.toJson(towerExtraVars)

  }

  stage('Run Integration Tests') {
    microcksTest(apiURL: params.MICROCKS_SERVER_URL,
                serviceId: params.MICROCKS_SERVICE_ID,
                testEndpoint: params.MICROCKS_TEST_ENDPOINT,
                runnerType: 'POSTMAN', verbose: 'true')
  }

  stage('Deploy API to prod') {
    // Tag the new build as "ready-for-prod"
    openshiftTag alias: 'false', destStream: params.OPENSHIFT_IMAGE_STREAM, srcTag: "${newVersion}",
                 destinationNamespace: params.OPENSHIFT_PROD_ENVIRONMENT, namespace: params.OPENSHIFT_BUILD_PROJECT,
                 srcStream: params.OPENSHIFT_IMAGE_STREAM, destTag: 'ready-for-prod', verbose: 'false'

    // Trigger a new deployment
    openshiftDeploy deploymentConfig: params.OPENSHIFT_DEPLOYMENT_CONFIG, namespace: params.OPENSHIFT_PROD_ENVIRONMENT

    // Deploy the API to 3scale
    ansibleTower towerServer: params.ANSIBLE_TOWER_SERVER,
                 inventory: params.ANSIBLE_PROD_INVENTORY,
                 jobTemplate: params.ANSIBLE_JOB_TEMPLATE,
                 extraVars: JsonOutput.toJson(towerExtraVars)
  }

}
