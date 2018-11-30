pipeline {
    agent none

    stages {
        stage('Build') {
            agent { label 'gradle' }
            steps {
                checkout scm
                sh "env"
                sh "gradle bootRepackage --stacktrace"
                sh 'jarFile=`ls build/libs | grep -v original` && mkdir -p ocp/deployments && cp build/libs/$jarFile ocp/deployments/'
            }
        }
        stage('Test') {
            agent { label 'gradle' }
            steps {
                parallel(
                    'check': {
                        sh "gradle check --stacktrace"
                    },
                    'echo': {
                        echo "ok in parallel"
                    }
                )
            }
        }
        stage('Build image') {
            agent { label 'gradle' }
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('dev') {
                            // create docker build
                            def buildTemplate = readYaml file: 'openshift/demo-docker-build.yml'
                            def resources = openshift.process(buildTemplate)
                            def buildCfg = openshift.apply(resources).narrow('bc')
                            // create app template
                            def template = readYaml file: 'openshift/demo-template.yml'
                            openshift.apply(openshift.process(template))

                            def buildSelector = buildCfg.startBuild('--from-dir=ocp')
                            timeout(5) {
                                buildSelector.untilEach(1) {
                                    return it.object().status.phase == "Complete"
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Dev') {
            agent { label 'gradle' }
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('dev') {
                            // TODO: replace it when https://github.com/openshift/jenkins-client-plugin/issues/84 will be solved
                            // deployment is triggered by imagestream here
                            openshiftVerifyDeployment depCfg: "demo", namespace: "dev"
                            sh "curl -L http://demo.dev.svc.cluster.local:8080/health"
                        }
                    }
                }
            }
        }
        stage('Staging') {
            agent { label 'gradle' }
            steps {
                script {
                    openshift.withCluster() {
                        openshift.tag("dev/demo:latest", "staging/demo:latest")
                        openshift.withProject('staging') {
                            def template = readYaml file: 'openshift/demo-template.yml'
                            openshift.apply(openshift.process(template, "-p", "VERSION=latest"))
                            // TODO: replace it when https://github.com/openshift/jenkins-client-plugin/issues/84 will be solved
                            openshiftVerifyDeployment depCfg: "demo", namespace: "staging"
                        }
                    }
                }
            }
        }
        stage('Approve') {
            agent none
            steps {
                input "Does the staging environment look ok ?"
            }
        }
        stage('Prod') {
            agent { label 'gradle' }
            steps {
                script {
                    openshift.withCluster() {
                        openshift.tag("staging/demo:latest", "prod/demo:latest")
                        openshift.withProject('prod') {
                            def template = readYaml file: 'openshift/demo-template.yml'
                            openshift.apply(openshift.process(template, "-p", "VERSION=latest"))
                            // TODO: replace it when https://github.com/openshift/jenkins-client-plugin/issues/84 will be solved
                            openshiftVerifyDeployment depCfg: "demo", namespace: "prod"
                        }
                    }
                }
            }
        }
    }
}
