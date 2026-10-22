pipeline {
    agent any

    parameters {
        choice(
            name: 'RELEASE_MODE',
            choices: ['NORMAL', 'FAULTY_ROLLOUT'],
            description: 'Choose a normal deployment or test automatic rollback'
        )
    }

    environment {
        DOCKER_USER = 'HeyItsArsh'
        IMAGE_NAME = 'road-hazard-portal'
        DEPLOYMENT = 'road-hazard-portal'
        NAMESPACE = 'default'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Parallel Validation') {
            parallel {
                stage('Validate HTML') {
                    steps {
                        sh '''
                            python3 -c "from pathlib import Path; p=Path('index.html'); s=p.read_text(); assert '<html' in s and '</html>' in s; assert 'complaints' in s; print('HTML validation passed')"
                        '''
                    }
                }

                stage('Validate Kubernetes YAML') {
                    steps {
                        sh '''
                            test -s deployment.yaml
                            grep -q 'kind: Deployment' deployment.yaml
                            grep -q 'kind: Service' deployment.yaml
                            echo "Deployment manifest validation passed"
                        '''
                    }
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: 'index.html,Dockerfile,deployment.yaml,Jenkinsfile',
                                 fingerprint: true
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${DOCKER_USER}/${IMAGE_NAME}:build-${BUILD_NUMBER} .
                    docker tag ${DOCKER_USER}/${IMAGE_NAME}:build-${BUILD_NUMBER} ${DOCKER_USER}/${IMAGE_NAME}:stable
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    sh '''
                        echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                        docker push ${DOCKER_USER}/${IMAGE_NAME}:build-${BUILD_NUMBER}
                        docker push ${DOCKER_USER}/${IMAGE_NAME}:stable
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy and Verify') {
            steps {
                withCredentials([
                    file(credentialsId: 'kuberconfig',
                         variable: 'KUBECONFIG_FILE')
                ]) {
                    script {
                        try {
                            sh '''
                                export KUBECONFIG="$KUBECONFIG_FILE"

                                kubectl apply -f deployment.yaml

                                if [ "$RELEASE_MODE" = "FAULTY_ROLLOUT" ]; then
                                    echo "Testing deliberate failure: image does not exist"
                                    IMAGE="${DOCKER_USER}/${IMAGE_NAME}:faulty-${BUILD_NUMBER}"
                                else
                                    IMAGE="${DOCKER_USER}/${IMAGE_NAME}:build-${BUILD_NUMBER}"
                                fi

                                kubectl set image deployment/${DEPLOYMENT} \
                                  road-hazard-portal="$IMAGE" \
                                  -n "$NAMESPACE"

                                kubectl rollout status deployment/${DEPLOYMENT} \
                                  -n "$NAMESPACE" --timeout=90s
                            '''

                            if (params.RELEASE_MODE == 'NORMAL') {
                                sh '''
                                    export KUBECONFIG="$KUBECONFIG_FILE"
                                    kubectl port-forward service/road-hazard-service 18080:80 \
                                      -n "$NAMESPACE" >/tmp/road-hazard-port-forward.log 2>&1 &
                                    PF_PID=$!
                                    trap 'kill "$PF_PID" 2>/dev/null || true' EXIT
                                    sleep 5
                                    curl --fail --silent --show-error \
                                      http://127.0.0.1:18080/ -o /dev/null
                                    echo "Portal HTTP verification passed"
                                '''
                            }
                        } catch (err) {
                            echo "Release failed. Rolling back to the previous stable revision."

                            sh '''
                                export KUBECONFIG="$KUBECONFIG_FILE"
                                kubectl rollout undo deployment/${DEPLOYMENT} \
                                  -n "$NAMESPACE"
                                kubectl rollout status deployment/${DEPLOYMENT} \
                                  -n "$NAMESPACE" --timeout=120s
                                kubectl get pods -n "$NAMESPACE"
                            '''

                            error("Deployment failed; rollback attempted. Inspect rollout logs.")
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Release mode: ${params.RELEASE_MODE}"
            echo "Pipeline finished. Review the stage logs and Kubernetes status."
        }
    }
}
