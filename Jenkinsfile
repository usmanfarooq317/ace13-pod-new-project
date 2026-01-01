pipeline {
    agent any

    environment {
        KUBECONFIG = credentials('kubeconfig-credentials-id') // Ensure this credential exists in Jenkins
    }

    stages {
        stage('Initialize Storage') {
            steps {
                script {
                    echo 'Applying PVC and Loader...'
                    sh 'kubectl apply -f pvc.yaml'
                    sh 'kubectl apply -f pvc-loader.yaml'
                    
                    echo 'Waiting for Loader Pod to be Ready...'
                    sh 'kubectl wait --for=condition=Ready pod/pvc-loader --timeout=60s'
                }
            }
        }

        stage('Upload BAR File') {
            steps {
                script {
                    echo 'Uploading BVSRegFix2.bar to PVC...'
                    sh 'kubectl cp BVSRegFix2.bar pvc-loader:/mnt/data/BVSRegFix2.bar'
                    
                    echo 'Cleaning up Loader Pod...'
                    sh 'kubectl delete pod pvc-loader --force --grace-period=0'
                }
            }
        }

        stage('Deploy Application') {
            steps {
                script {
                    echo 'Applying Deployment and HPA...'
                    sh 'kubectl apply -f deployment.yaml'
                    sh 'kubectl apply -f hpa.yaml'
                    
                    echo 'Waiting for Rollout...'
                    sh 'kubectl rollout status deployment/ace-deployment'
                }
            }
        }

        stage('Verify & Stream Logs') {
            steps {
                script {
                    echo 'Streaming logs from the new pods...'
                    // Get pod names
                    def pods = sh(script: "kubectl get pods -l app=ace-app -o jsonpath='{.items[*].metadata.name}'", returnStdout: true).trim().split(' ')
                    
                    // Stream logs from the first pod found to show specific steps
                    if (pods.length > 0) {
                        def podName = pods[0]
                        echo "Showing logs for pod: ${podName}"
                        // Use --follow=false to just print current logs, or true to stream for a while
                        // Since deployment is Ready, initial logs should be there. 
                        // Using -f to follow until user aborts or we time out, but for pipeline we usually just dump 
                        // or run a background job. Here we just dump recent logs to show the steps.
                        sh "kubectl logs ${podName}"
                    } else {
                        echo 'No pods found to stream logs from.'
                    }
                }
            }
        }
    }
}
