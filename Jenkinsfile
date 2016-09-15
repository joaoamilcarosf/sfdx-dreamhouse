#!groovy
node {
	def SERIAL = System.currentTimeMillis()
	def BRANCH = env.BRANCH_NAME
	def BUILD_NUMBER=env.BUILD_NUMBER
	def ORG_DEF_JSON_FILE="dhci-org-def-${BRANCH}-${BUILD_NUMBER}-pr.json"
	def SUB_ORG_DEF_JSON_FILE="dhci-sub-org-def-${BRANCH}-${BUILD_NUMBER}-pr.json"
	def RUN_ARTIFACT_DIR="tests/${BUILD_NUMBER}"
	def SFDC_USERNAME="ci-${BRANCH}-${SERIAL}-pr@dhci.com"
	def SUB_SFDC_USERNAME="ci-sub-${BRANCH}-${SERIAL}-pr@dhci.com"
	def HUB_ORG=env.HUB_ORG_DH
	def HUB_KEY=env.HUB_KEY_FILE_PATH
	def SFDC_HOST = env.SFDC_HOST
	def CONNECTED_APP_CONSUMER_KEY=env.CONNECTED_APP_CONSUMER_KEY
	def CONNECTED_APP_CALLBACK_URL=env.CONNECTED_APP_CALLBACK_URL
	def SIGN_UP_EMAIL=env.SIGN_UP_EMAIL
	def API_VERSION=env.API_VERSION
	def toolbelt = tool 'toolbelt'

	stage('checkout source') {
		// when running in multi-branch job, one must issue this command
		checkout scm
	}

	// to save runtime - it is probably best that toolbelt be installed as a Jenkins setup step
	// rather than installing each time a job is run
	stage('Install Toolbelt') {
		// check to see if install of toolbelt is necessary
		rc = sh returnStatus: true, script: "${toolbelt}/heroku force --help"
		if (rc != 0) {
			rc = sh returnStatus: true, script: "${toolbelt}/heroku plugins:install salesforce-alm-dev"
			if (rc != 0) {
				error 'toolbelt install failed'
			}
			rc = sh returnStatus: true, script: "${toolbelt}/heroku force --help"
			if (rc != 0) {
				error 'toolbelt install verification failed'
			}
		}
	}

	stage('Create Scratch Org') {
		// modify workspace file to point to correct Salesforce App Server
		sh "mkdir -p ${RUN_ARTIFACT_DIR}"

		config = sprintf('''{
	    		"sfdcLoginUrl" : "%s",
	    		"apiVersion" : "%s",
	    		"defaultArtifact" : "helloWorld"
					}''', SFDC_HOST, API_VERSION)

		orgDef = sprintf('''{
				    "Company": "Force.com CI",
				    "Country": "US",
				    "LastName": "Developer",
				    "SignupEmail": "%s",
				    "Username": "%s",
				    "Edition": "editionDefinition/force.com/DeveloperEdition",
				    "OrgPreferences" : {"S1DesktopEnabled" : true},
				    "ConnectedAppConsumerKey" : "%s",
				    "ConnectedAppCallbackUrl" : "%s"
		     	}''', SIGN_UP_EMAIL, SFDC_USERNAME, CONNECTED_APP_CONSUMER_KEY, CONNECTED_APP_CALLBACK_URL)

		writeFile encoding: 'utf-8', file: 'config.json', text: config

		writeFile encoding: 'utf-8', file: "${RUN_ARTIFACT_DIR}/${ORG_DEF_JSON_FILE}", text: orgDef

		rc = sh returnStatus: true, script: "${toolbelt}/heroku force:org:authorize -i ${CONNECTED_APP_CONSUMER_KEY} -u ${HUB_ORG} -f ${HUB_KEY} -y debug"
		if (rc != 0) {
			error 'hub org authorization failed'
		}

		rc = sh returnStatus: true, script: "${toolbelt}/heroku force:org:create -f ${RUN_ARTIFACT_DIR}/${ORG_DEF_JSON_FILE} -t test -y debug"
		if (rc != 0) {
			error 'org creation failed'
		}
	}

	stage('Push To Test Org') {
		rc = sh returnStatus: true, script: "${toolbelt}/heroku force:src:push --all --targetname ${SFDC_USERNAME} -y debug"
		if (rc != 0) {
			error 'push all failed'
		}
		// assign permset
		rc = sh returnStatus: true, script: "${toolbelt}/heroku force:permset:assign --targetname ${SFDC_USERNAME} --name DreamHouse -y debug"
		if (rc != 0) {
			error 'push all failed'
		}
	}

	stage('Run Apex Test') {
		timeout(time: 120, unit: 'SECONDS') {
			rc = sh returnStatus: true, script: "${toolbelt}/heroku force:apex:test --testlevel RunLocalTests --testartifactdir ${RUN_ARTIFACT_DIR} --reporter tap --targetname ${SFDC_USERNAME} -y debug"
			if (rc != 0) {
				error 'apex test run failed'
			}
		}
	}

	// test runner is hanging consistently
	//    stage('Run Test via Test Runner') {
	//    	timeout(time: 60, unit: 'SECONDS') {
	//    		withEnv(["RUN_ARTIFACT_DIR=${RUN_ARTIFACT_DIR}"]) {
	//				rc = sh returnStatus: true, script: "${toolbelt}/heroku force:test --config ./testrunner/simple.json -y debug"
	//        		if (rc != 0) {
	//            		error 'apex test run failed'
	//        		}
	//        	}
	//    	}
	//	}

	stage('collect results') {
		junit keepLongStdio: true, testResults: 'tests/**/*-junit.xml'
	}
}
