pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: node-build
    image: node:26-bookworm
    command:
    - cat
    tty: true
  - name: docker
    image: docker:24-dind
    securityContext:
      privileged: true
    env:
    - name: "added value"
      value: "12"
    - name: testtinf
      value: "hello"
    - name: DOCKER_TLS_CERTDIR
      value: ""
    volumeMounts:
    - name: registry-ca
      mountPath: /etc/docker/certs.d/registry.registry.svc.cluster.local:5000
      readOnly: true
  volumes:
  - name: registry-ca
    secret:
      secretName: registry-tls
      items:
      - key: tls.crt
        path: ca.crt
"""
        }
    }

    tools {
        nodejs 'NodeJS'
    }

    environment {
        LOCAL_REGISTRY     = 'registry.registry.svc.cluster.local:5000'
        LOCAL_REGISTRY_CREDS = 'my-local-registry'        // Jenkins credential ID
        IMAGE_NAME         = 'iquant-app'
        IMAGE_FULL         = "registry.registry.svc.cluster.local:5000/iquant-app"
        image_tag          = "staging"
    }

    stages {

        stage('Checkout Github') {
            steps {
                git branch: 'staging',
                    credentialsId: 'salim-git1',
                    url: 'https://github.com/salimep/jenkin-1.git'
            }
        }

        stage('Install node dependencies') {
            steps {
                container('node-build') {
                    sh 'npm install'
                }
            }
        }

        stage('Test Code') {
            steps {
                container('node-build') {
                    sh 'npm test'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh """
                        docker build -t ${IMAGE_FULL}:latest \
                                     -t ${IMAGE_FULL}:${image_tag} .
                    """
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                container('docker') {
                    sh """
                        # Install Trivy
                        apk add --no-cache curl
                        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
                            | sh -s -- -b /usr/local/bin

                        # Run scan against local image
                        trivy image \
                            --severity HIGH,CRITICAL \
                            --no-progress \
                            --format table \
                            -o trivy-scan-report.txt \
                            ${IMAGE_FULL}:latest
                    """
                }
            }
        }

        stage('Push Image to Local Registry') {
            steps {
                container('docker') {
                    script {
                        // Login to local registry using stored Jenkins credentials
                        withCredentials([usernamePassword(
                            credentialsId: "${LOCAL_REGISTRY_CREDS}",
                            usernameVariable: 'REG_USER',
                            passwordVariable: 'REG_PASS'
                        )]) {
                            sh """
                                docker login ${LOCAL_REGISTRY} \
                                    -u \$REG_USER \
                                    -p \$REG_PASS

                                docker push ${IMAGE_FULL}:latest
                                docker push ${IMAGE_FULL}:${image_tag}

                                docker logout ${LOCAL_REGISTRY}
                            """
                        }
                    }
                }
            }
        }

        stage('Verify Image in Registry') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                        credentialsId: "${LOCAL_REGISTRY_CREDS}",
                        usernameVariable: 'REG_USER',
                        passwordVariable: 'REG_PASS'
                    )]) {
                        sh """
                            # List images in local registry
                            curl -sk -u \$REG_USER:\$REG_PASS \
                                https://${LOCAL_REGISTRY}/v2/_catalog

                            # List tags for this image
                            curl -sk -u \$REG_USER:\$REG_PASS \
                                https://${LOCAL_REGISTRY}/v2/${IMAGE_NAME}/tags/list
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
    steps {
        container('docker') {
            withKubeConfig([credentialsId: 'kube-config']) {
                sh """
                    # Install kubectl
                    apk add --no-cache curl
                    curl -LO "https://dl.k8s.io/release/\$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                    chmod +x kubectl && mv kubectl /usr/local/bin/

                    sed -i 's|IMAGE_TAG|${image_tag}|g' deployment.yaml
                    kubectl apply -f deployment.yaml
                    kubectl rollout status deployment/node-app-deployment \
                     --timeout=180s
                """
            }
        }
    }
}
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'trivy-scan-report.txt', allowEmptyArchive: true
        }
        success {
            echo "✅ Image pushed to local registry: ${IMAGE_FULL}:${image_tag}"
        }
        failure {
            echo '❌ Build failed. Check logs.'
        }
    }
}
