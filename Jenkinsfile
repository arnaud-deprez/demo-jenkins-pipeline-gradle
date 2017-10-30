pipeline {
    agent { label 'gradle' }

    stages {
        stage('Build') {
            steps {
                git url: "https://github.com/arnaud-deprez/demo-jenkins-pipeline-gradle.git"
                sh "gradle bootRepackage --stacktrace"
                sh 'jarFile=`ls build/libs | grep -v original` && mkdir -p ocp/deployments && cp build/libs/$jarFile ocp/deployments/'
            }
        }
        stage('Test') {
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
            steps {
                sh "oc process -f openshift/demo-docker-build.yml | oc apply -n dev -f -"
                sh "oc process -f openshift/demo-template.yml | oc apply -n dev -f -"
                sh "oc start-build demo --from-dir=ocp -n dev"
                openshiftVerifyBuild bldCfg: "demo", waitTime: "20", waitUnit: "min", namespace: "dev"
            }
        }
        stage('Dev') {
            steps {
                // deployment is triggered by imagestream here
                openshiftVerifyDeployment depCfg: "demo", namespace: "dev"
                sh "curl -L http://demo.dev.svc.cluster.local:8080/health"
            }
        }
        stage('Staging') {
            steps {
                openshiftTag namespace: "dev", srcStream: "demo", srcTag: "latest", destinationNamespace: "staging", destStream: "demo", destTag: "latest"
                // deployment is triggered by imagestream here
                sh "oc process -f openshift/demo-template.yml -p VERSION=latest | oc apply -n staging -f -"
                openshiftVerifyDeployment depCfg: "demo", namespace: "staging"
            }
        }
        stage('Prod') {
            steps {
                input "Does the staging environment look ok ?"
                openshiftTag namespace: "staging", srcStream: "demo", srcTag: "latest", destinationNamespace: "prod", destStream: "demo", destTag: "latest"
                // deployment is triggered by imagestream here
                sh "oc process -f openshift/demo-template.yml -p VERSION=latest | oc apply -n prod -f -"
                openshiftVerifyDeployment depCfg: "demo", namespace: "prod"
            }
        }
    }
}