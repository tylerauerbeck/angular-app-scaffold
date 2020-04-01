pipeline {

    agent {
        // label "" also could have been 'agent any' - that has the same meaning.
        label "master"
    }

    environment {
        // GLobal Vars
        PROJECT_NAMESPACE = "summit-tekton"
        APP_NAME = "angular-fe"
        NODE_ENV="test"
        E2E_TEST_ROUTE = "oc get route/${APP_NAME} --template='{{.spec.host}}' -n ${PROJECT_NAMESPACE}".execute().text.minus("'").minus("'")

        JENKINS_TAG = "${JOB_NAME}.${BUILD_NUMBER}".replace("/", "-")
        JOB_NAME = "${JOB_NAME}".replace("/", "-")

        SONAR_SCANNER_HOME = tool "sonar-scanner-tool"
    }

    // The options directive is for configuration that applies to the whole job.
    options {
        buildDiscarder(logRotator(numToKeepStr: '50', artifactNumToKeepStr: '1'))
        timeout(time: 15, unit: 'MINUTES')
    }

    stages {
        stage("node-build") {
            agent {
                node {
                    label "jenkins-slave-npm"
                }
            }
            steps {
                sh 'printenv'

                echo '### Install deps ###'
                sh 'npm install'

                echo '### Install deps ###'
                sh 'npm run clean'

                echo '### Running linter ###'
                sh 'npm run lint'

                echo '### Running tests ###'
                sh 'npm run test:ci'
                sh 'npm run e2e:ci'

                // echo '### Running sonar scanner ###'
                // script {
                //     def scannerHome = tool 'sonar-scanner-tool';
                //     withSonarQubeEnv('sonar') {
                //         sh "${scannerHome}/bin/sonar-runner"
                //      }
                //  }
                echo '### Running build ###'
                sh '''
                    npm run build
                '''

                // echo '### Running pact tests ###'
                // sh 'npm run pact:test'
                // sh '''
                //     if [ -z "${NODE_ENV}" ];then
                //     echo "We are in feature branch, not publishing Pact contract"
                //     else
                //     npm run pact:publish
                //     fi
                // '''
                // echo '### Checking of Pact Broker returns deployable for our versions ###'
                // script {
                //     try {
                //         sh '''
                //             if [ -z "${NODE_ENV}" ];then
                //             echo "We are in feature branch, not verifying Pact contract"
                //             else
                //             npm run pact:verify
                //             fi
                //         '''
                //     } catch (exc) {
                //         echo 'Pact Contract not verified!'
                //         currentBuild.result = 'UNSTABLE'
                //     }
                // }
                echo '### Packaging App for Nexus ###'
                sh 'npm run package'
                sh 'npm run publish'
            }
            // Post can be used both on individual stages and for the entire build.
        }

        stage("node-bake") {
            agent {
                node {
                    label "master"
                }
            }
            when {
                expression { GIT_BRANCH ==~ /(.*master|.*develop)/ }
            }
            steps {
                echo '### Get Binary from Nexus ###'
                sh  '''
                        rm -rf package-contents*
                        curl -v -f http://${NEXUS_CREDS}@${NEXUS_SERVICE_HOST}:${NEXUS_SERVICE_PORT}/repository/labs-static/com/redhat/fe/${JENKINS_TAG}/package-contents.zip -o package-contents.zip
                        unzip package-contents.zip
                    '''
                echo '### Create Linux Container Image from package ###'
                sh  '''
                        oc project ${PIPELINES_NAMESPACE} # probs not needed
                        oc patch bc ${APP_NAME} -p "{\\"spec\\":{\\"output\\":{\\"to\\":{\\"kind\\":\\"ImageStreamTag\\",\\"name\\":\\"${APP_NAME}:${JENKINS_TAG}\\"}}}}"
                        oc start-build ${APP_NAME} --from-dir=package-contents/ --follow
                    '''
            }
        }

        stage("node-deploy") {
            agent {
                node {
                    label "master"
                }
            }
            when {
                expression { GIT_BRANCH ==~ /(.*master|.*develop)/ }
            }
            steps {
                echo '### tag image for namespace ###'
                sh  '''
                    oc project ${PROJECT_NAMESPACE}
                    oc tag ${PIPELINES_NAMESPACE}/${APP_NAME}:${JENKINS_TAG} ${PROJECT_NAMESPACE}/${APP_NAME}:${JENKINS_TAG}
                    '''
                echo '### set env vars and image for deployment ###'
                sh '''
                    oc set image dc/${APP_NAME} ${APP_NAME}=docker-registry.default.svc:5000/${PROJECT_NAMESPACE}/${APP_NAME}:${JENKINS_TAG}
                    oc rollout latest dc/${APP_NAME}
                '''
                echo '### Verify OCP Deployment ###'
                openshiftVerifyDeployment depCfg: env.APP_NAME,
                    namespace: env.PROJECT_NAMESPACE,
                    replicaCount: '1',
                    verbose: 'false',
                    verifyReplicaCount: 'true',
                    waitTime: '',
                    waitUnit: 'sec'
            }
            post {
                success {
                    build job: 'system-test', parameters: [[$class: 'StringParameterValue', name: 'PROJECT_NAMESPACE', value: "${PROJECT_NAMESPACE}" ],[$class: 'StringParameterValue', name: 'JENKINS_TAG', value: "${JENKINS_TAG}"]], wait: false
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts "**"
        }
    }
}
