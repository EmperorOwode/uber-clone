pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID= credentials('account_id')
        AWS_DEFAULT_REGION="us-east-1" 
        AWS_ACCESS_KEY_ID = credentials('aws_access_key_id')
        AWS_SECRET_ACCESS_KEY = credentials('aws_secret_access_key')
    }    
    stages {
        stage('Run Terrascan') {
            steps {
                    // Run Terrascan and save the JSON output
                script {
                    def scanStatus = sh(
                        script: '''
                        docker run --rm -v /var/lib/jenkins/workspace/eks_deployment:/iac tenable/terrascan:latest scan -d /iac/EKS_Terraform -o json > terrascan_output.json
                        ''',
                        returnStatus: true
                    )
      
                    // Archive Terrascan results
                    archiveArtifacts artifacts: 'terrascan_output.json', allowEmptyArchive: true
                    
                    // Parse JSON output
                    def jsonContent = readFile('terrascan_output.json')
                    def parsedJSON = new groovy.json.JsonSlurper().parseText(jsonContent)
                    
                    // Count medium and high severity violations
                    def mediumViolations = parsedJSON.results.violations.findAll { it.severity == 'MEDIUM' }.size()
                    def highViolations = parsedJSON.results.violations.findAll { it.severity == 'HIGH' }.size()

                    // Fail pipeline if medium or high severity vulnerabilities exist
                    if (mediumViolations > 0 || highViolations > 0) {
                        error("Terrascan found ${mediumViolations} medium and ${highViolations} high severity vulnerabilities. Check terrascan_output.json for details.")
                    } else {
                        echo "No critical vulnerabilities found. Proceeding with Terraform commands."
                    }
                }
            }
        }
        stage('Terraform Init') {
            steps {
                sh '''
                cd /var/lib/jenkins/workspace/eks_deployment/EKS_Terraform
                terraform init -reconfigure -input=false
                '''
            }
        }
        stage('Terraform Validate') {
            steps {
                sh '''
                cd /var/lib/jenkins/workspace/eks_deployment/EKS_Terraform
                terraform validate
                '''
            }
        }
        stage('Terraform Apply') {
            steps {
                // Automatically approve the apply step
                sh '''
                cd /var/lib/jenkins/workspace/eks_deployment/EKS_Terraform
                terraform ${action} -auto-approve
                '''
            }
        }
    }
    post {
        always {
            echo 'Pipeline execution completed.'
        }
        failure {
            echo 'Pipeline failed. Please check the logs for details.'
        }
    }
}
