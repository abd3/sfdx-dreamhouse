#!groovy
import groovy.json.JsonSlurperClassic
node {

	def BUILD_NUMBER=env.BUILD_NUMBER
	def RUN_ARTIFACT_DIR="tests/${BUILD_NUMBER}"
	def SFDC_USERNAME
	
	def HUB_ORG=env.HUB_ORG_DH
	def SFDC_HOST = env.SFDC_HOST_DH
	def JWT_KEY_CRED_ID = env.JWT_CRED_ID_DH
	def CONNECTED_APP_CONSUMER_KEY=env.CONNECTED_APP_CONSUMER_KEY_DH

	def toolbelt = tool 'toolbelt'

	stage('checkout source') {
		// when running in multi-branch job, one must issue this command
		checkout scm
	}

	withCredentials([file(credentialsId: JWT_KEY_CRED_ID, variable: 'jwt_key_file')]) {

		stage('Create Scratch Org') {

			rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:org:authorize -i ${CONNECTED_APP_CONSUMER_KEY} -u ${HUB_ORG} -f ${jwt_key_file} -y debug"
			if (rc != 0) { error 'hub org authorization failed' }

			// need to pull out assigned username
			rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:org:create -f config/workspace-scratch-def.json -j -t test -y debug"
			printf rmsg
			def jsonSlurper = new JsonSlurperClassic()
			def robj = jsonSlurper.parseText(rmsg)
			if (robj.status != "ok") { error 'org creation failed: ' + robj.message }
			SFDC_USERNAME=robj.username
			robj = null

		}

		stage('Push To Test Org') {
			rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:src:push --all --username ${SFDC_USERNAME} -y debug"
			if (rc != 0) {
				error 'push all failed'
			}
			// assign permset
			rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:permset:assign --username ${SFDC_USERNAME} --name DreamHouse -y debug"
			if (rc != 0) {
				error 'push all failed'
			}
		}

		stage('Run Apex Test') {
			sh "mkdir -p ${RUN_ARTIFACT_DIR}"
			timeout(time: 120, unit: 'SECONDS') {
				rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:apex:test --testlevel RunLocalTests --outputdir ${RUN_ARTIFACT_DIR} --reporter tap --username ${SFDC_USERNAME} -y debug"
				if (rc != 0) {
					error 'apex test run failed'
				}
			}
		}

		stage('collect results') {
			junit keepLongStdio: true, testResults: 'tests/**/*-junit.xml'
		}
	}
}
