pipeline {
    agent any

    environment {
        PROJECT_ID   = credentials('gcp-project-id')
        REGION       = credentials('gke-region')
        CLUSTER_NAME = credentials('gke-cluster-name')
        
        GCP_CRED_ID = 'gcp-service-account-key' 
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

//test comment