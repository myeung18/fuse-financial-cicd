#!/usr/bin/env groovy
import groovy.json.JsonOutput
import groovy.json.JsonSlurper
import javax.net.ssl.HostnameVerifier
import javax.net.ssl.HttpsURLConnection
import javax.net.ssl.SSLContext
import javax.net.ssl.TrustManager
import javax.net.ssl.X509TrustManager
import java.security.SecureRandom
import java.net.URLEncoder




def gitRepo = params.GIT_REPO
def gitBranch = params.GIT_BRANCH != null && params.GIT_BRANCH != "" ? params.GIT_BRANCH : "master"



node('maven') {



	def towerExtraVars = [
		git_repo: gitRepo,
		git_branch: gitBranch,
		//threescale_cicd_api_backend_hostname: params.OPENSHIFT_SERVICE_NAME + ":8080",
		threescale_cicd_api_backend_hostname: params.API_URL,
		threescale_cicd_openapi_smoketest_operation: params.ANSIBLE_SMOKE_TEST_OPREATION,
		threescale_cicd_api_backend_scheme: "http",
		threescale_cicd_api_base_system_name: params.OPENSHIFT_SERVICE_NAME,
		threescale_cicd_validate_openapi: false,
		openapi_file: params.SWAGGER_FILE_NAME
	]



	stage('CreateRouteInside3scale') {

		catchError {

			sh "oc process -f "+params.API_CAST_ROUTE_TEMPLATE_FILE+" -p BASE_NAME="+params.OPENSHIFT_SERVICE_NAME+" -p MAJOR_VERSION="+params.MAJOR_VERSION+" -p WILDCARD_DOMAIN="+params.WILDCARD_DOMAIN+" | oc create -f - -n "+params.THREESCALE_OPENSHIFT_PROJECT

		}

	}


	stage('Deploy API with Ansible to 3scale') {


		// Deploy the API to 3scale
		ansibleTower towerServer: params.ANSIBLE_TOWER_SERVER,
		inventory: params.ANSIBLE_TEST_INVENTORY,
		jobTemplate: params.ANSIBLE_JOB_TEMPLATE,
		extraVars: JsonOutput.toJson(towerExtraVars)

	}



	stage ('Start wth Groovy') {
		def userInput = input( id: "userInput", message: "Let's Deploy API with Groovy?", parameters: [
			[$class: "TextParameterDefinition", defaultValue: "Comments?", description: "comments", name: "comments"]
		])
		print 'promotionCheck'
	}

	stage('Create Service with Grovy') {

		create3scaleService(params.THREESCALE_URL,params.API_TOKEN,
				"https://raw.githubusercontent.com/redhatHameed/fuse-financial-cicd/master/openapi-spec.json",
				"3scalefuse")

	}

}



def create3scaleService(
		String adminBaseUrl,
		String token,
		String backendServiceSwaggerEndpoint,
		String serviceSystemName) {


	def jsonSlurper = new JsonSlurper()
	println('Fetching service swagger json...')
	def swaggerDoc = jsonSlurper.parseText(new URL(backendServiceSwaggerEndpoint).getText())

	println('Creating Service...')
	def activeDocSpecCreateUrl = "${adminBaseUrl}/admin/api/services.json"
	def name = swaggerDoc.info.title != null ? swaggerDoc.info.title : serviceSystemName
	def data = "access_token=${token}&name=${name}&system_name=${serviceSystemName}"
	println('Data...'+data)
	println('CreateUrl...'+activeDocSpecCreateUrl);

	makeRequestwithBody(activeDocSpecCreateUrl, data, 'POST')

}


def update3scaleActiveDoc(
		String adminBaseUrl,
		String token,
		String productionRoute,
		String backendServiceSwaggerEndpoint,
		String serviceSystemName) {

	def activeDocSpecListUrl = "${adminBaseUrl}/admin/api/active_docs.json?access_token=${token}"
	def servicesEndpoint = "${adminBaseUrl}/admin/api/services.json?access_token=${token}"

	// instantiate JSON parser
	def jsonSlurper = new JsonSlurper()
	println('Fetching service swagger json...')
	// fetch new swagger doc from service
	def swaggerDoc = jsonSlurper.parseText(new URL(backendServiceSwaggerEndpoint).getText())
	def services =  jsonSlurper.parseText(new URL(servicesEndpoint).getText()).services
	def service = (services.find { it.service['system_name'] == serviceSystemName }).service
	// get the auth config for this service, so we can add the correct auth params to the swagger doc
	def proxyEndpoint = "${adminBaseUrl}/admin/api/services/${service.id}/proxy.json?access_token=${token}"
	def proxyConfig =  jsonSlurper.parseText(new URL(proxyEndpoint).getText()).proxy
	swaggerDoc.host = productionRoute;
	// is the credential in a header or query param?
	def credentialsLocation = proxyConfig['credentials_location'] == 'headers' ? 'header' : 'query'
	// for each method on each path, add the param for credentials
	def paths = swaggerDoc.paths.keySet() as List
	paths.each { path ->
		def methods = swaggerDoc.paths[path].keySet() as List
		methods.each {
			if (!(swaggerDoc.paths[path][it].parameters)) {
				swaggerDoc.paths[path][it].parameters = []
			}
			swaggerDoc.paths[path][it].parameters.push([
				in: credentialsLocation,
				name: proxyConfig['auth_user_key'],
				description: 'User authorization key',
				required: true,
				type: 'string'
			])
		}
	}
	// get all existing docs on 3scale
	println('Fetching uploaded Active Docs...')
	def activeDocs =  jsonSlurper.parseText(new URL(activeDocSpecListUrl).getText())['api_docs']
	// find the one matching the correct service (or not)
	def activeDoc = activeDocs.find { it['api_doc']['system_name'] == serviceSystemName }
	if ( activeDoc ) {
		println('Found Active Doc for ' + serviceSystemName)
		def activeDocId = activeDoc['api_doc'].id
		// update our swagger doc, now containing the correct host and auth params to 3scale
		def activeDocSpecUpdateUrl = "${adminBaseUrl}/admin/api/active_docs/${activeDocId}.json"
		def name = swaggerDoc.info.title != null ? swaggerDoc.info.title : serviceSystemName
		def data = "access_token=${token}&body=${URLEncoder.encode(JsonOutput.toJson(swaggerDoc), 'UTF-8')}&skip_swagger_validations=false"
		makeRequestwithBody(activeDocSpecUpdateUrl, data, 'PUT')
	} else {
		println('Active Docs for ' + serviceSystemName + ' not found in 3scale. Creating a new Active Doc.')
		// post our new swagger doc, now containing the correct host and auth params to 3scale
		def activeDocSpecCreateUrl = "${adminBaseUrl}/admin/api/active_docs.json"
		def name = swaggerDoc.info.title != null ? swaggerDoc.info.title : serviceSystemName
		def data = "access_token=${token}&name=${name}&system_name=${serviceSystemName}&body=${URLEncoder.encode(JsonOutput.toJson(swaggerDoc), 'UTF-8')}&skip_swagger_validations=false"
		makeRequestwithBody(activeDocSpecCreateUrl, data, 'POST')
	}
}



def makeRequestwithBody(url, body, method) {
	def post = new URL(url).openConnection();
	post.setRequestMethod(method)
	post.setDoOutput(true)
	post.setRequestProperty('Content-Type', 'application/x-www-form-urlencoded')
	post.setRequestProperty('Accept', 'application/json')
	post.getOutputStream().write(body.getBytes('UTF-8'))
	def responseCode = post.getResponseCode();
	if (responseCode != 200 && responseCode != 201) {
		println('Failed to update/create . HTTP response: ' + responseCode)
		assert false
	} else {
		println('updated/created successfully!')
	}
}





