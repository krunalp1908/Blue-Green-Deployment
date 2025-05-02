@Library('Shared') _

pipeline {
    agent any

    tools {
        maven 'maven3'
    }

    environment {
        IMAGE_NAME = "krunalp19/bankapp"
        TAG = "${params.DOCKER_TAG}"
        KUBE_NAMESPACE = 'webapps'
        SCANNER_HOME = tool 'Sonar'
    }

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')
    }

    stages {

        stage("Validate Parameters") {
            steps {
                script {
                    if (params.DOCKER_TAG.trim() == '') {
                        error("DOCKER_TAG must be provided.")
                    }
                    if (params.DEPLOY_ENV != params.DOCKER_TAG) {
                        error("DEPLOY_ENV and DOCKER_TAG must match for consistency.")
                    }
                }
            }
        }

        stage("Git Checkout") {
            steps {
                script {
                    code_checkout("https://github.com/krunalp1908/Blue-Green-Deployment.git", "main")
                }
            }
        }

        stage("Compile") {
            steps {
                sh 'mvn compile'
            }
        }

        stage("Testing") {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }

        stage("Trivy: Filesystem Scan") {
            steps {
                sh "trivy fs --format table -o fs.html ."
            }
        }

        stage("SonarQube Code Analysis") {
            steps {
                withSonarQubeEnv('Sonar') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=GCBank \
                        -Dsonar.projectKey=GCBank \
                        -Dsonar.sources=. \
                        -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }

        stage("SonarQube Quality Gate") {
            steps {
                timeout(time: 1, unit: "HOURS") {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage("Build") {
            steps {
                sh 'mvn clean package -DskipTests=true'
            }
        }

        stage("Publish Artifact") {
            steps {
                withMaven(globalMavenSettingsConfig: 'my-setting-kp', maven: 'maven3', traceability: true) {
                    sh 'mvn deploy -DskipTests=true'
                }
            }
        }

        stage("Docker: Build Image") {
            steps {
                script {
                    docker_build("bankapp", TAG, "krunalp19")
                }
            }
        }

        stage("Trivy: Image Scan") {
            steps {
                script {
                    try {
                        sh "trivy image --format table -o image_scan.html ${IMAGE_NAME}:${TAG}"
                    } catch (err) {
                        echo "Trivy scan failed: ${err}"
                    }
                }
            }
        }

        stage("Docker: Push to Docker Hub") {
            steps {
                script {
                    docker_push("bankapp", TAG, "krunalp19")
                }
            }
        }

        stage("Deploy MySQL Deployment and Service") {
            steps {
                script {
                    withKubeConfig(
                        caCertificate: '',
                        clusterName: 'devopsshack-cluster',
                        contextName: '',
                        credentialsId: 'k8s-token',
                        namespace: KUBE_NAMESPACE,
                        restrictKubeConfigAccess: false,
                        serverUrl: 'https://C8320CCE62DBDFDB046D58C4889D58CB.gr7.us-east-1.eks.amazonaws.com'
                    ) {
                        sh "kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE}"
                    }
                }
            }
        }

        stage("Deploy SVC-APP") {
            steps {
                script {
                    withKubeConfig(
                        caCertificate: '',
                        clusterName: 'devopsshack-cluster',
                        contextName: '',
                        credentialsId: 'k8s-token',
                        namespace: KUBE_NAMESPACE,
                        restrictKubeConfigAccess: false,
                        serverUrl: 'https://C8320CCE62DBDFDB046D58C4889D58CB.gr7.us-east-1.eks.amazonaws.com'
                    ) {
                        sh """
                            if ! kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}; then
                                kubectl apply -f bankapp-service.yml -n ${KUBE_NAMESPACE}
                            fi
                        """
                    }
                }
            }
        }

        stage("Deploy to Kubernetes") {
            steps {
                script {
                    def deploymentFile = params.DEPLOY_ENV == 'blue' ? 'app-deployment-blue.yml' : 'app-deployment-green.yml'
                    withKubeConfig(
                        caCertificate: '',
                        clusterName: 'devopsshack-cluster',
                        contextName: '',
                        credentialsId: 'k8s-token',
                        namespace: KUBE_NAMESPACE,
                        restrictKubeConfigAccess: false,
                        serverUrl: 'https://C8320CCE62DBDFDB046D58C4889D58CB.gr7.us-east-1.eks.amazonaws.com'
                    ) {
                        sh "kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}"
                    }
                }
            }
        }

        stage("Switch Traffic Between Blue & Green Environment") {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                script {
                    def newEnv = params.DEPLOY_ENV
                    withKubeConfig(
                        caCertificate: '',
                        clusterName: 'devopsshack-cluster',
                        contextName: '',
                        credentialsId: 'k8s-token',
                        namespace: KUBE_NAMESPACE,
                        restrictKubeConfigAccess: false,
                        serverUrl: 'https://C8320CCE62DBDFDB046D58C4889D58CB.gr7.us-east-1.eks.amazonaws.com'
                    ) {
                        sh "kubectl patch service bankapp-service -p '{\"spec\": {\"selector\": {\"app\": \"bankapp\", \"version\": \"${newEnv}\"}}}' -n ${KUBE_NAMESPACE}"
                    }
                    echo "✅ Traffic has been switched to the ${newEnv} environment."
                }
            }
        }

        stage("Verify Deployment") {
            steps {
                script {
                    def verifyEnv = params.DEPLOY_ENV
                    withKubeConfig(
                        caCertificate: '',
                        clusterName: 'devopsshack-cluster',
                        contextName: '',
                        credentialsId: 'k8s-token',
                        namespace: KUBE_NAMESPACE,
                        restrictKubeConfigAccess: false,
                        serverUrl: 'https://C8320CCE62DBDFDB046D58C4889D58CB.gr7.us-east-1.eks.amazonaws.com'
                    ) {
                        sh """
                            kubectl get pods -l version=${verifyEnv} -n ${KUBE_NAMESPACE}
                            kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
