pipeline {
  agent any
 
  environment {
    IO_POC_PROJECT_NAME = "insecure-bank"
    POLARIS_PROJECT_NAME = "akshayme-synp/insecure-bank"
    BLACKDUCK_PROJECT_NAME = "insecure-bank"
    BLACKDUCK_PROJECT_VERSION = "1.0.0"
    POLARIS_ACCESS_TOKEN = credentials('polaris-token')
    BLACKDUCK_ACCESS_TOKEN = credentials('BlackDuck-AuthToken')
    IO_ACCESS_TOKEN = credentials('IO-AUTH-TOKEN')
    GITHUB_USERNAME = "akshayme-synp"
    GITHUB_ACCESS_TOKEN = credentials('Github-AuthToken')
    IS_SAST_ENABLED = "false"
    IS_SCA_ENABLED = "false"
    IS_DAST_ENABLED = "false"
    IS_IMAGE_SCAN_ENABLED = "false"
    IS_CODE_REVIEW_ENABLED = "false"
    IS_PEN_TESTING_ENABLED = "false"
  }
 
  stages {
    stage('Build') {
      steps {
        sh 'mvn -e clean package -DskipTests'
      }
    }
    stage('IO Prescription') {
      steps {
        echo "Getting IO Prescription"
        sh '''
          rm -fr prescription.sh
          wget "https://raw.githubusercontent.com/synopsys-sig/io-artifacts/${WORKFLOW_CLIENT_VERSION}/prescription.sh"
          sed -i -e 's/\r$//' prescription.sh
          chmod a+x prescription.sh
          ./prescription.sh \
          --stage="IO" \
          --persona="devsecops" \
          --io.url="${IO_URL}" \
          --io.token="${IO_ACCESS_TOKEN}" \
          --manifest.type="json" \
          --project.name="${IO_POC_PROJECT_NAME}" \
          --workflow.url="${WORKFLOW_URL}" \
          --workflow.version="${WORKFLOW_CLIENT_VERSION}" \
          --scm.type="github" \
          --scm.owner="${GITHUB_USERNAME}" \
          --scm.repo.name="${IO_POC_PROJECT_NAME}" \
          --scm.branch.name="master" \
          --github.username="${GITHUB_USERNAME}" \
          --github.token="${GITHUB_ACCESS_TOKEN}" \
          --polaris.project.name="${POLARIS_PROJECT_NAME}" \
          --polaris.url="${POLARIS_SERVER_URL}" \
          --polaris.token="${POLARIS_ACCESS_TOKEN}" \
          --blackduck.project.name="${BLACKDUCK_PROJECT_NAME}:${BLACKDUCK_PROJECT_VERSION}" \
          --blackduck.url="${BLACKDUCK_URL}" \
          --blackduck.api.token="${BLACKDUCK_ACCESS_TOKEN}" \
          --jira.enable="false" \
          --IS_SAST_ENABLED="${IS_SAST_ENABLED}" \
          --IS_SCA_ENABLED="${IS_SCA_ENABLED}" \
          --IS_DAST_ENABLED="${IS_DAST_ENABLED}"
        '''
        sh 'mv result.json io-presciption.json'
        sh '''
          echo "==================================== IO Risk Score =======================================" > io-risk-score.txt
          echo "Business Criticality Score - $(jq -r '.riskScoreCard.bizCriticalityScore' io-presciption.json)" >> io-risk-score.txt
          echo "Data Class Score - $(jq -r '.riskScoreCard.dataClassScore' io-presciption.json)" >> io-risk-score.txt
          echo "Access Score - $(jq -r '.riskScoreCard.accessScore' io-presciption.json)" >> io-risk-score.txt
          echo "Open Vulnerability Score - $(jq -r '.riskScoreCard.openVulnScore' io-presciption.json)" >> io-risk-score.txt
          echo "Change Significance Score - $(jq -r '.riskScoreCard.changeSignificanceScore' io-presciption.json)" >> io-risk-score.txt
          export bizScore=$(jq -r '.riskScoreCard.bizCriticalityScore' io-presciption.json | cut -d'/' -f2)
          export dataScore=$(jq -r '.riskScoreCard.dataClassScore' io-presciption.json | cut -d'/' -f2)
          export accessScore=$(jq -r '.riskScoreCard.accessScore' io-presciption.json | cut -d'/' -f2)
          export vulnScore=$(jq -r '.riskScoreCard.openVulnScore' io-presciption.json | cut -d'/' -f2)
          export changeScore=$(jq -r '.riskScoreCard.changeSignificanceScore' io-presciption.json | cut -d'/' -f2)
          echo -n "Total Score - " >> io-risk-score.txt && echo "$bizScore + $dataScore + $accessScore + $vulnScore + $changeScore" | bc >> io-risk-score.txt
        '''
        sh 'cat io-risk-score.txt'
        sh '''
          echo "IS_SAST_ENABLED = $(jq -r '.security.activities.sast.enabled' io-presciption.json)" > io-prescription.txt
          echo "IS_SCA_ENABLED = $(jq -r '.security.activities.sca.enabled' io-presciption.json)" >> io-prescription.txt
          echo "IS_DAST_ENABLED = $(jq -r '.security.activities.dast.enabled' io-presciption.json)" >> io-prescription.txt
          echo "IS_IMAGE_SCAN_ENABLED = $(jq -r '.security.activities.imageScan.enabled' io-presciption.json)" >> io-prescription.txt
          echo "IS_CODE_REVIEW_ENABLED = $(jq -r '.security.activities.sastplusm.enabled' io-presciption.json)" >> io-prescription.txt
          echo "IS_PEN_TESTING_ENABLED = $(jq -r '.security.activities.dastplusm.enabled' io-presciption.json)" >> io-prescription.txt
        '''
        sh 'cat io-prescription.txt'
      }
    }
    stage('SAST - Coverity') {
      steps {
        echo "Stage - Coverity on Polaris"
        sh '''
          IS_SAST_ENABLED=$(jq -r '.security.activities.sast.enabled' io-presciption.json)
          echo "IS_SAST_ENABLED = ${IS_SAST_ENABLED}"
          if [ ${IS_SAST_ENABLED} = "true" ]; then
            echo "Running Coverity on Polaris based on IO Precription"
            rm -fr /tmp/polaris
            wget -q ${POLARIS_SERVER_URL}/api/tools/polaris_cli-linux64.zip
            unzip -j polaris_cli-linux64.zip -d /tmp
            /tmp/polaris --persist-config --co project.name="${POLARIS_PROJECT_NAME}" --co project.branch="master" --co capture.build.buildCommands="null" --co capture.build.cleanCommands="null" --co capture.fileSystem="null" --co capture.coverity.autoCapture="enable"  configure
            /tmp/polaris analyze -w
          else
            echo "Skipping Coverity on Polaris based on IO Precription"
          fi
          '''
      }
    }
    stage('SCA - Blackduck') {
      steps {
        echo "Stage - Blackduck"
        sh '''
          IS_SCA_ENABLED=$(jq -r '.security.activities.sca.enabled' io-presciption.json)
          echo "IS_SCA_ENABLED = ${IS_SCA_ENABLED}"
          if [ ${IS_SCA_ENABLED} = "true" ]; then
            echo "Running Blackduck based on IO Precription"
            rm -fr /tmp/detect7.sh
            curl -s -L https://detect.synopsys.com/detect7.sh > /tmp/detect7.sh
            bash /tmp/detect7.sh --blackduck.url="${BLACKDUCK_URL}" --blackduck.api.token="${BLACKDUCK_ACCESS_TOKEN}" --detect.project.name="${BLACKDUCK_PROJECT_NAME}" --detect.project.version.name=${BLACKDUCK_PROJECT_VERSION} --blackduck.trust.cert=true
          else
            echo "Skipping Blackduck based on IO Precription"
          fi
          '''
      }
    }
    stage('IO Workflow') {
      steps {
        echo "Preparing to run IO Workflow Engine"
        sh '''
          IS_SAST_ENABLED=$(jq -r '.security.activities.sast.enabled' io-presciption.json)
          IS_SCA_ENABLED=$(jq -r '.security.activities.sca.enabled' io-presciption.json)
          ./prescription.sh \
          --stage="WORKFLOW" \
          --persona="devsecops" \
          --io.url="${IO_URL}" \
          --io.token="${IO_ACCESS_TOKEN}" \
          --manifest.type="json" \
          --project.name="${IO_POC_PROJECT_NAME}" \
          --workflow.url="${WORKFLOW_URL}" \
          --workflow.version="${WORKFLOW_CLIENT_VERSION}" \
          --polaris.project.name="${POLARIS_PROJECT_NAME}" \
          --polaris.url="${POLARIS_SERVER_URL}" \
          --polaris.token="${POLARIS_ACCESS_TOKEN}" \
          --blackduck.project.name="${BLACKDUCK_PROJECT_NAME}:${BLACKDUCK_PROJECT_VERSION}" \
          --blackduck.url="${BLACKDUCK_URL}" \
          --blackduck.api.token="${BLACKDUCK_ACCESS_TOKEN}" \
          --jira.enable="false" \
          --IS_SAST_ENABLED="${IS_SAST_ENABLED}" \
          --IS_SCA_ENABLED="${IS_SCA_ENABLED}" \
          --IS_DAST_ENABLED="${IS_DAST_ENABLED}"
        '''
        echo "Running IO Workflow Engine"
        sh '''
          java -jar WorkflowClient.jar --workflowengine.url="${WORKFLOW_URL}" --io.manifest.path=synopsys-io.json
        '''
      }
    }
    stage('Schedule Manual Activities') {
      steps {
        echo "Check for Scheduling of Manual Actviities"
        echo "Manual Code Review"
        sh '''
          IS_CODE_REVIEW_ENABLED=$(jq -r '.security.activities.sastplusm.enabled' io-presciption.json)
          echo "IS_CODE_REVIEW_ENABLED = ${IS_CODE_REVIEW_ENABLED}"
          if [ ${IS_CODE_REVIEW_ENABLED} = "true" ]; then
            echo "Sending Notification for Manual Code Review based on IO Precription"
            # Put code to send notification here
          else
            echo "Skipping Manual Code Review based on IO Precription"
          fi
          '''
        echo "Manual Penetration Testing"
        sh '''
          IS_PEN_TESTING_ENABLED=$(jq -r '.security.activities.dastplusm.enabled' io-presciption.json)
          echo "IS_PEN_TESTING_ENABLED = ${IS_PEN_TESTING_ENABLED}"
          if [ ${IS_PEN_TESTING_ENABLED} = "true" ]; then
            echo "Sending Notification for Manual Penetration Testing based on IO Precription"
            # Put code to send notification here
          else
            echo "Skipping Manual Penetration Testing based on IO Precription"
          fi
          '''
      }
    }
    stage('Break the Build') {
      steps {
        echo "add Build Breaker parts here"
        sh '''
          echo "Breaker Status - $(jq -r '.breaker.status' wf-output.json)"
          # Put code to break the build here
          IS_BREAKER_STATUS_ENABLED=$(jq -r '.breaker.status' wf-output.json)
          echo "Breaker Status - $(IS_BREAKER_STATUS_ENABLED)"
          if [ ${IS_BREAKER_STATUS_ENABLED} = "true" ]; then
              echo "$(jq -r '.breaker.criteria[0]' wf-output.json)"
          fi
        '''
      }
    }
    stage('Clean Workspace') {
      steps {
        cleanWs()
      }
    }
  }
}
