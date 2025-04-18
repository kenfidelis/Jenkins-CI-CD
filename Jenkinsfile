// Jenkins Shared Library Import
@Library('my-org-shared-library') _

// Pipeline Definition
pipeline {
    agent {
        kubernetes {
            yaml """
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: maven
                    image: maven:3.8.6-openjdk-11
                    command: ['cat']
                    tty: true
                  - name: docker
                    image: docker:20.10.16
                    command: ['cat']
                    tty: true
                    volumeMounts:
                    - name: docker-sock
                      mountPath: /var/run/docker.sock
                  - name: terraform
                    image: hashicorp/terraform:1.2.0
                    command: ['cat']
                    tty: true
                  volumes:
                  - name: docker-sock
                    hostPath:
                      path: /var/run/docker.sock
            """
        }
    }
    
    // Pipeline options
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 60, unit: 'MINUTES')
        ansiColor('xterm')
        timestamps()
        disableConcurrentBuilds()
    }
    
    // Parameters for the pipeline run
    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'test', 'staging', 'prod'], description: 'Deployment Environment')
        string(name: 'VERSION', defaultValue: '', description: 'Version to deploy (leave empty for latest)')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run automated tests')
        booleanParam(name: 'FEATURE_FLAG_NEW_UI', defaultValue: false, description: 'Enable new UI features')
        text(name: 'RELEASE_NOTES', defaultValue: '', description: 'Release notes for this deployment')
    }

    // Environment variables
    environment {
        APP_NAME = 'advanced-application'
        SERVICE_NAME = "${APP_NAME}-service"
        DOCKER_REGISTRY = 'registry.example.com'
        ARTIFACT_REPO = 'https://artifacts.example.com/repository'
        BUILD_VERSION = "${env.BUILD_NUMBER}-${getShortCommitHash()}"
        SONAR_PROJECT_KEY = 'org.example:advanced-application'
        DEPLOY_TIMESTAMP = sh(script: 'date +%Y%m%d%H%M%S', returnStdout: true).trim()
        
        // Credentials
        DOCKER_CREDENTIALS = credentials('docker-registry-credentials')
        AWS_CREDENTIALS = credentials('aws-credentials')
        SONAR_TOKEN = credentials('sonarqube-token')
        
        // Feature flags
        ENABLE_NEW_UI = "${params.FEATURE_FLAG_NEW_UI}"
    }
    
    stages {
        stage('Initialize') {
            steps {
                script {
                    // Display build information
                    echo "Building ${APP_NAME} version ${BUILD_VERSION}"
                    echo "Deployment target: ${params.DEPLOY_ENV}"
                    
                    // Set environment-specific variables
                    loadEnvironmentConfig(params.DEPLOY_ENV)
                    
                    // Setup notification channels
                    notifyBuildStart()
                }
            }
        }
        
        stage('Checkout') {
            steps {
                checkout scm
                
                // Git submodules if needed
                sh 'git submodule update --init --recursive'
                
                script {
                    // Store git details for later use
                    env.GIT_COMMIT_MSG = sh(script: 'git log -1 --pretty=%B ${GIT_COMMIT}', returnStdout: true).trim()
                    env.GIT_AUTHOR = sh(script: 'git log -1 --pretty=%an ${GIT_COMMIT}', returnStdout: true).trim()
                }
            }
        }
        
        stage('Build') {
            steps {
                container('maven') {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            mvn clean package \
                                -Dbuild.version=${BUILD_VERSION} \
                                -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                                -Dsonar.login=${SONAR_TOKEN} \
                                -Pfeature.new-ui=${ENABLE_NEW_UI} \
                                sonar:sonar
                        """
                    }
                    
                    // Archive the build artifacts
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
            post {
                success {
                    // Generate code quality reports
                    recordIssues enabledForFailure: true, tools: [checkStyle(), spotBugs()]
                    
                    // Publish test results
                    junit 'target/surefire-reports/*.xml'
                    
                    // Publish code coverage
                    jacoco execPattern: 'target/jacoco.exec'
                }
            }
        }
        
        stage('Security Scan') {
            parallel {
                stage('OWASP Dependency Check') {
                    steps {
                        container('maven') {
                            sh 'mvn org.owasp:dependency-check-maven:check'
                        }
                    }
                    post {
                        always {
                            dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
                        }
                    }
                }
                
                stage('SonarQube Quality Gate') {
                    steps {
                        timeout(time: 5, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }
            }
        }
        
        stage('Build Container') {
            steps {
                container('docker') {
                    sh """
                        echo ${DOCKER_CREDENTIALS_PSW} | docker login ${DOCKER_REGISTRY} -u ${DOCKER_CREDENTIALS_USR} --password-stdin
                        docker build \
                            --build-arg VERSION=${BUILD_VERSION} \
                            --build-arg ENABLE_NEW_UI=${ENABLE_NEW_UI} \
                            -t ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_VERSION} \
                            -t ${DOCKER_REGISTRY}/${APP_NAME}:latest .
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_VERSION}
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}:latest
                    """
                }
                
                // Scan container for vulnerabilities
                sh 'trivy image ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_VERSION} --exit-code 1 --severity HIGH,CRITICAL'
                
                // Sign the container image
                sh 'cosign sign --key ${COSIGN_KEY} ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_VERSION}'
            }
        }
        
        stage('Deploy Infrastructure') {
            when {
                expression { return params.DEPLOY_ENV != 'prod' || params.DEPLOY_ENV == 'prod' && isApproved() }
            }
            steps {
                container('terraform') {
                    withCredentials([file(credentialsId: 'terraform-key', variable: 'TF_KEY_FILE')]) {
                        sh """
                            cd terraform/${params.DEPLOY_ENV}
                            terraform init
                            terraform workspace select ${params.DEPLOY_ENV} || terraform workspace new ${params.DEPLOY_ENV}
                            terraform plan -var "app_version=${BUILD_VERSION}" -var "timestamp=${DEPLOY_TIMESTAMP}" -out=tfplan
                            terraform apply -auto-approve tfplan
                        """
                    }
                }
                
                script {
                    // Store infrastructure outputs for later use
                    env.CLUSTER_ENDPOINT = sh(script: 'terraform output -json | jq -r .cluster_endpoint.value', returnStdout: true).trim()
                    env.DATABASE_URL = sh(script: 'terraform output -json | jq -r .database_url.value', returnStdout: true).trim()
                }
            }
        }
        
        stage('Deploy Application') {
            when {
                expression { return params.DEPLOY_ENV != 'prod' || params.DEPLOY_ENV == 'prod' && isApproved() }
            }
            steps {
                script {
                    // Generate Kubernetes manifests with Helm
                    sh """
                        helm template ${APP_NAME} ./helm \
                            --set image.repository=${DOCKER_REGISTRY}/${APP_NAME} \
                            --set image.tag=${BUILD_VERSION} \
                            --set environment=${params.DEPLOY_ENV} \
                            --set featureFlags.newUI=${ENABLE_NEW_UI} \
                            --set replica.count=${getReplicaCount(params.DEPLOY_ENV)} \
                            --set database.url=${DATABASE_URL} \
                            --namespace=${params.DEPLOY_ENV} \
                            > kubernetes/manifests.yaml
                    """
                    
                    // Apply proper deployment strategy based on environment
                    if (params.DEPLOY_ENV == 'prod') {
                        // Blue-green deployment for production
                        blueGreenDeploy('kubernetes/manifests.yaml', params.DEPLOY_ENV)
                    } else if (params.DEPLOY_ENV == 'staging') {
                        // Canary deployment for staging
                        canaryDeploy('kubernetes/manifests.yaml', params.DEPLOY_ENV)
                    } else {
                        // Direct deployment for dev/test
                        sh "kubectl apply -f kubernetes/manifests.yaml --namespace=${params.DEPLOY_ENV}"
                    }
                    
                    // Wait for deployment to complete
                    sh "kubectl rollout status deployment/${APP_NAME} --namespace=${params.DEPLOY_ENV} --timeout=300s"
                }
            }
        }
        
        stage('Run Tests') {
            when {
                expression { return params.RUN_TESTS }
            }
            parallel {
                stage('API Tests') {
                    steps {
                        sh "newman run postman/collection.json -e postman/${params.DEPLOY_ENV}-env.json --reporters cli,junit,html --reporter-junit-export results/newman-report.xml"
                    }
                    post {
                        always {
                            junit 'results/newman-report.xml'
                        }
                    }
                }
                
                stage('UI Tests') {
                    steps {
                        sh "cypress run --config-file cypress/${params.DEPLOY_ENV}.config.js"
                    }
                    post {
                        always {
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'cypress/reports',
                                reportFiles: 'index.html',
                                reportName: 'Cypress Test Report'
                            ])
                        }
                    }
                }
                
                stage('Performance Tests') {
                    when {
                        expression { return params.DEPLOY_ENV == 'staging' || params.DEPLOY_ENV == 'prod' }
                    }
                    steps {
                        sh "k6 run k6/performance-test.js --env ENV=${params.DEPLOY_ENV}"
                    }
                    post {
                        always {
                            perfReport 'k6/results.xml'
                        }
                    }
                }
            }
        }
        
        stage('Smoke Test') {
            steps {
                script {
                    def endpoint = getServiceEndpoint(params.DEPLOY_ENV)
                    sh "curl -f ${endpoint}/health || exit 1"
                    sh "curl -f ${endpoint}/version | grep ${BUILD_VERSION} || exit 1"
                }
            }
        }
        
        stage('Compliance Check') {
            when {
                expression { return params.DEPLOY_ENV == 'staging' || params.DEPLOY_ENV == 'prod' }
            }
            steps {
                script {
                    // Run compliance scans
                    sh "inspec exec compliance/${params.DEPLOY_ENV}-profile -t aws://-- --format junit > compliance-results.xml"
                }
            }
            post {
                always {
                    junit 'compliance-results.xml'
                }
            }
        }
        
        stage('Finalize Release') {
            when {
                expression { return params.DEPLOY_ENV == 'prod' }
            }
            steps {
                script {
                    // Create a release tag in Git
                    sh """
                        git tag -a v${BUILD_VERSION} -m "Release ${BUILD_VERSION}"
                        git push origin v${BUILD_VERSION}
                    """
                    
                    // Update release documentation
                    updateReleaseNotes(params.RELEASE_NOTES, BUILD_VERSION)
                    
                    // Store artifacts for long-term retention
                    archiveArtifactsToS3()
                }
            }
        }
    }
    
    post {
        success {
            script {
                notifyDeploymentSuccess(params.DEPLOY_ENV, BUILD_VERSION)
                updateStatusBoard('SUCCESS')
                
                // For production deployments, create JIRA tickets for monitoring
                if (params.DEPLOY_ENV == 'prod') {
                    createMonitoringTicket(BUILD_VERSION)
                }
            }
        }
        failure {
            script {
                notifyDeploymentFailure(params.DEPLOY_ENV, BUILD_VERSION)
                updateStatusBoard('FAILURE')
                
                // Auto-rollback for non-prod environments
                if (params.DEPLOY_ENV != 'prod') {
                    rollbackDeployment(params.DEPLOY_ENV)
                }
            }
        }
        always {
            // Archive logs and reports
            archiveArtifacts artifacts: 'results/**/*', allowEmptyArchive: true
            
            // Clean workspace
            cleanWs()
        }
    }
}

// Helper functions
def getShortCommitHash() {
    return sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
}

def getReplicaCount(environment) {
    switch(environment) {
        case 'dev':
            return 1
        case 'test':
            return 2
        case 'staging':
            return 3
        case 'prod':
            return 5
        default:
            return 1
    }
}

def loadEnvironmentConfig(environment) {
    def config = readYaml file: "config/${environment}.yaml"
    config.each { key, value ->
        env."${key}" = value
    }
}

def isApproved() {
    try {
        timeout(time: 24, unit: 'HOURS') {
            def userInput = input(
                id: 'confirmDeployment',
                message: 'Deploy to Production?',
                parameters: [
                    [
                        $class: 'BooleanParameterDefinition',
                        defaultValue: false,
                        description: 'Deploy to production?',
                        name: 'confirm'
                    ]
                ]
            )
            return userInput
        }
    } catch(err) {
        return false
    }
}

def blueGreenDeploy(manifest, namespace) {
    // Create the green environment
    sh """
        sed -i 's/name: ${APP_NAME}/name: ${APP_NAME}-green/g' ${manifest}
        kubectl apply -f ${manifest} --namespace=${namespace}
        kubectl rollout status deployment/${APP_NAME}-green --namespace=${namespace} --timeout=300s
    """
    
    // Run validation tests against green
    sh "pytest validation-tests/ --base-url=https://${APP_NAME}-green.example.com"
    
    // Switch traffic if tests pass
    sh """
        kubectl patch service ${APP_NAME} -p '{"spec":{"selector":{"app":"${APP_NAME}-green"}}}' --namespace=${namespace}
        sleep 60
    """
    
    // Remove old blue deployment
    sh "kubectl delete deployment ${APP_NAME}-blue --namespace=${namespace} || true"
    
    // Rename green to blue for next deployment
    sh """
        kubectl patch deployment ${APP_NAME}-green -p '{"metadata":{"name":"${APP_NAME}-blue"},"spec":{"selector":{"matchLabels":{"app":"${APP_NAME}-blue"}},"template":{"metadata":{"labels":{"app":"${APP_NAME}-blue"}}}}}' --namespace=${namespace}
    """
}

def canaryDeploy(manifest, namespace) {
    // Deploy canary with 10% traffic
    sh """
        sed -i 's/name: ${APP_NAME}/name: ${APP_NAME}-canary/g' ${manifest}
        sed -i 's/replicas: .*/replicas: 1/g' ${manifest}
        kubectl apply -f ${manifest} --namespace=${namespace}
        kubectl rollout status deployment/${APP_NAME}-canary --namespace=${namespace} --timeout=300s
    """
    
    // Update ingress to split traffic
    sh """
        kubectl patch ingress ${APP_NAME}-ingress --type=json -p='[{"op": "add", "path": "/spec/rules/0/http/paths/0/backend/service/name", "value":"${APP_NAME}-canary", "weight": 10}]' --namespace=${namespace}
    """
    
    // Monitor canary for 10 minutes
    sh "canary-monitor --app=${APP_NAME}-canary --duration=600 --namespace=${namespace}"
    
    // If canary is healthy, deploy to main
    sh """
        sed -i 's/name: ${APP_NAME}-canary/name: ${APP_NAME}/g' ${manifest}
        sed -i 's/replicas: .*/replicas: ${getReplicaCount(namespace)}/g' ${manifest}
        kubectl apply -f ${manifest} --namespace=${namespace}
        kubectl rollout status deployment/${APP_NAME} --namespace=${namespace} --timeout=300s
    """
    
    // Remove canary
    sh "kubectl delete deployment ${APP_NAME}-canary --namespace=${namespace}"
    
    // Restore ingress
    sh """
        kubectl patch ingress ${APP_NAME}-ingress --type=json -p='[{"op": "replace", "path": "/spec/rules/0/http/paths/0/backend/service/name", "value":"${APP_NAME}"}]' --namespace=${namespace}
    """
}

def getServiceEndpoint(environment) {
    switch(environment) {
        case 'dev':
            return "https://dev.${APP_NAME}.example.com"
        case 'test':
            return "https://test.${APP_NAME}.example.com"
        case 'staging':
            return "https://staging.${APP_NAME}.example.com"
        case 'prod':
            return "https://${APP_NAME}.example.com"
        default:
            return "https://dev.${APP_NAME}.example.com"
    }
}

def notifyBuildStart() {
    slackSend(
        channel: getNotificationChannel(params.DEPLOY_ENV),
        color: '#FFFF00',
        message: "STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})\nDeploying ${APP_NAME}:${BUILD_VERSION} to ${params.DEPLOY_ENV}"
    )
}

def notifyDeploymentSuccess(environment, version) {
    slackSend(
        channel: getNotificationChannel(environment),
        color: 'good',
        message: "SUCCESS: '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})\n${APP_NAME}:${version} has been successfully deployed to ${environment}\nEndpoint: ${getServiceEndpoint(environment)}"
    )
    
    // Send email for production deployments
    if (environment == 'prod') {
        emailext(
            subject: "Production deployment successful: ${APP_NAME} ${version}",
            body: """
                <p>The deployment of ${APP_NAME} version ${version} to production was successful.</p>
                <p>Build URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                <p>Release Notes:</p>
                <pre>${params.RELEASE_NOTES}</pre>
            """,
            to: 'team@example.com, stakeholders@example.com',
            mimeType: 'text/html'
        )
    }
}

def notifyDeploymentFailure(environment, version) {
    slackSend(
        channel: getNotificationChannel(environment),
        color: 'danger',
        message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})\nDeployment of ${APP_NAME}:${version} to ${environment} failed"
    )
    
    // Always send email on failures
    emailext(
        subject: "Deployment FAILED: ${APP_NAME} ${version} to ${environment}",
        body: """
            <p>The deployment of ${APP_NAME} version ${version} to ${environment} has failed.</p>
            <p>Build URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
            <p>Console Output: <a href="${env.BUILD_URL}/console">${env.BUILD_URL}/console</a></p>
        """,
        to: 'devops@example.com, team@example.com',
        mimeType: 'text/html'
    )
}

def getNotificationChannel(environment) {
    switch(environment) {
        case 'prod':
            return '#deployments-prod'
        case 'staging':
            return '#deployments-staging'
        default:
            return '#deployments-dev'
    }
}

def updateStatusBoard(status) {
    sh """
        curl -X POST https://statusboard.example.com/api/update \
            -H 'Content-Type: application/json' \
            -d '{"app":"${APP_NAME}","env":"${params.DEPLOY_ENV}","version":"${BUILD_VERSION}","status":"${status}","timestamp":"${DEPLOY_TIMESTAMP}"}'
    """
}

def createMonitoringTicket(version) {
    def jiraBody = [
        fields: [
            project: [key: 'OPS'],
            summary: "Monitor deployment of ${APP_NAME} version ${version}",
            description: "Monitor the deployment of ${APP_NAME} version ${version} for 24 hours after release.\n\nDeployment details:\n- Build: ${env.BUILD_URL}\n- Release Notes: ${params.RELEASE_NOTES}",
            issuetype: [name: 'Task'],
            assignee: [name: 'oncall'],
            priority: [name: 'Major'],
            labels: ['monitoring', 'deployment', APP_NAME]
        ]
    ]
    
    def response = httpRequest(
        contentType: 'APPLICATION_JSON',
        httpMode: 'POST',
        requestBody: groovy.json.JsonOutput.toJson(jiraBody),
        url: 'https://jira.example.com/rest/api/2/issue',
        authentication: 'jira-api-token'
    )
    
    def ticket = readJSON text: response.content
    echo "Created monitoring ticket: ${ticket.key}"
}

def rollbackDeployment(environment) {
    echo "Starting rollback for ${APP_NAME} in ${environment}"
    
    // Get previous successful version
    def previousVersion = sh(
        script: "kubectl get deployment ${APP_NAME} -n ${environment} -o jsonpath='{.metadata.annotations.lastStableVersion}'",
        returnStdout: true
    ).trim()
    
    if (previousVersion) {
        sh """
            # Update deployment to previous version
            kubectl set image deployment/${APP_NAME} ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${previousVersion} -n ${environment}
            kubectl rollout status deployment/${APP_NAME} -n ${environment} --timeout=300s
        """
        
        echo "Rolled back to version ${previousVersion}"
        slackSend(
            channel: getNotificationChannel(environment),
            color: 'warning',
            message: "ROLLBACK: ${APP_NAME} in ${environment} has been rolled back to version ${previousVersion}"
        )
    } else {
        echo "No previous version found, cannot rollback automatically"
        slackSend(
            channel: getNotificationChannel(environment),
            color: 'danger',
            message: "ROLLBACK FAILED: Cannot rollback ${APP_NAME} in ${environment} - no previous version found"
        )
    }
}

def updateReleaseNotes(notes, version) {
    withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
        sh """
            curl -X POST \
                -H "Authorization: token ${GITHUB_TOKEN}" \
                -H "Accept: application/vnd.github.v3+json" \
                https://api.github.com/repos/organization/${APP_NAME}/releases \
                -d '{"tag_name":"v${version}","name":"Release ${version}","body":"${notes}","draft":false,"prerelease":false}'
        """
    }
}

def archiveArtifactsToS3() {
    withAWS(credentials: 'aws-credentials', region: 'us-west-2') {
        s3Upload(
            bucket: 'app-releases',
            path: "${APP_NAME}/${BUILD_VERSION}/",
            includePathPattern: 'target/*.jar'
        )
    }
}
