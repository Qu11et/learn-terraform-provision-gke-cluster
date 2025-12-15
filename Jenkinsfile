pipeline {
    agent any

    environment {
        PROJECT_ID   = credentials('gcp-project-id')
        REGION       = credentials('gke-region')
        CLUSTER_NAME = credentials('gke-cluster-name')
        
        GCP_CRED_ID = 'gcp-service-account-key'
        REPORT_FILE  = "report.json" 
    }

    triggers {
        githubPush() 
    }

    stages {
        stage('Terraform Apply') {
            steps {
                script {
                    withCredentials([file(credentialsId: GCP_CRED_ID, variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                        sh 'terraform init'
                        sh 'terraform plan -out=tfplan'
                        sh 'terraform apply -auto-approve tfplan'
                    }
                }
            }
        }

        stage('Connect to GKE') {
            steps {
                withCredentials([file(credentialsId: GCP_CRED_ID, variable: 'GC_KEY')]) {
                    script {
                        sh 'gcloud auth activate-service-account --key-file=$GC_KEY'
                        sh "gcloud config set project ${PROJECT_ID}"
                        
                        sh "gcloud container clusters get-credentials ${CLUSTER_NAME} --region ${REGION}"
                    }
                }
            }
        }
        
        stage('Security Audit (CIS Benchmark)') {
            steps {
                script {
                    echo "--- Deploying Kube-bench Job ---"
                    
                    sh 'kubectl delete job kube-bench-gke --ignore-not-found=true'

                    sh 'kubectl apply -f kube-bench-job.yaml'

                    try {
                        sh 'kubectl wait --for=condition=complete --timeout=120s job/kube-bench-gke'
                    } catch (Exception e) {
                        error "Kube-bench job timed out! Something is wrong with the cluster."
                    }

                    sh "kubectl logs job/kube-bench-gke > ${REPORT_FILE}"
                    
                    echo "Report saved to ${REPORT_FILE}"
                }
            }
        }

        stage('Analyze Security Results') {
            steps {
                script {
                    echo "--- Analyzing CIS Report ---"
                    
                    def failCount = sh(
                        script: "cat ${REPORT_FILE} | jq '[.. | select(.status? == \"FAIL\")] | length'", 
                        returnStdout: true
                    ).trim().toInteger()

                    echo "Total Security Failures Found: ${failCount}"
                    
                    archiveArtifacts artifacts: REPORT_FILE, fingerprint: true

                    if (failCount > 0) {
                        echo "Details of Failures:"
                        sh "cat ${REPORT_FILE} | jq '.. | select(.status? == \"FAIL\") | {ID: .test_number, Desc: .test_desc, Remediation: .remediation}'"
                        
                        error "⛔ SECURITY CHECK FAILED: Found ${failCount} violations. Deployment blocked!"
                    } else {
                        echo "✅ Security Check Passed! Cluster is compliant."
                    }
                }
            }
        }

        stage('Analyze Security Results') {
            steps {
                script {
                    echo "--- Analyzing CIS Report ---"
                    
                    // --- CẤU HÌNH CÁC ID MUỐN BỎ QUA ---
                    // Đây là danh sách các lỗi bạn chấp nhận rủi ro
                    def IGNORED_IDS_LOGIC = 'select(.test_number != "4.1.5" and .test_number != "5.6.7")'
                    
                    // 1. Đếm số lỗi THỰC TẾ (Đã trừ các ID bị ignore)
                    // Logic: Lấy tất cả FAIL -> Lọc bỏ 4.1.5 và 5.6.7 -> Đếm số còn lại
                    def failCount = sh(
                        script: "cat ${env.REPORT_FILE} | jq '[.. | select(.status? == \"FAIL\") | ${IGNORED_IDS_LOGIC}] | length'", 
                        returnStdout: true
                    ).trim().toInteger()

                    echo "Total Critical Failures (After Whitelist): ${failCount}"
                    
                    // Lưu báo cáo gốc
                    archiveArtifacts artifacts: env.REPORT_FILE, fingerprint: true

                    // 2. In ra cảnh báo cho các lỗi bị Ignore (Để không bị quên)
                    echo "--- [WARNING] The following errors were IGNORED by policy ---"
                    sh "cat ${env.REPORT_FILE} | jq '.. | select(.status? == \"FAIL\") | select(.test_number == \"4.1.5\" or .test_number == \"5.6.7\") | {ID: .test_number, Desc: .test_desc}'"

                    // 3. QUYẾT ĐỊNH (GATEKEEPER)
                    if (failCount > 0) {
                        echo "Details of NEW Critical Failures:"
                        // In chi tiết các lỗi MỚI (không nằm trong whitelist)
                        sh "cat ${env.REPORT_FILE} | jq '.. | select(.status? == \"FAIL\") | ${IGNORED_IDS_LOGIC} | {ID: .test_number, Desc: .test_desc, Remediation: .remediation}'"
                        
                        error "⛔ SECURITY CHECK FAILED: Found ${failCount} UNACCEPTED violations. Deployment blocked!"
                    } else {
                        echo "✅ Security Check Passed! (With accepted risks)."
                    }
                }
            }
        }

        stage('Deploy Application (Simulated)') {
            steps {
                script {
                    echo "=============================================="
                    echo "🚀 CONGRATULATIONS! ALL SECURITY CHECKS PASSED"
                    echo "=============================================="
                    echo "Deploying application to production..."
                    echo "..."
                    echo "Application Deployed Successfully."
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed.'
        }
        failure {
            echo 'Pipeline failed. Please check the logs.'
        }
    }    
}

//test comment 2