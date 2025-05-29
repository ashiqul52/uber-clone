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
                script {
                    def scanStatus = sh(
                        script: '''
                        docker run --rm -v /var/lib/jenkins/workspace/eks_deployment/EKS_Terraform:/iac tenable/terrascan:latest scan -d /iac -o json > terrascan_output.json
                        ''',
                        returnStatus: true
                    )
                    
                    archiveArtifacts artifacts: 'terrascan_output.json', allowEmptyArchive: true

                    def jsonContent = readFile('terrascan_output.json').trim()

                    if (!jsonContent) {
                        error("Terrascan did not return valid output. Directory may be empty or contain no Terraform files.")
                    }

                    def parsedJSON = new groovy.json.JsonSlurper().parseText(jsonContent)

                    def violations = parsedJSON.results.violations ?: []
                    def mediumViolations = violations.findAll { it.severity == 'MEDIUM' }.size()
                    def highViolations = violations.findAll { it.severity == 'HIGH' }.size()

                    if (mediumViolations > 0 || highViolations > 0) {
                        error("Terrascan found ${mediumViolations} medium and ${highViolations} high severity vulnerabilities. Check terrascan_output.json for details.")
                    } else {
                        echo "✅ No critical vulnerabilities found. Proceeding with Terraform steps."
                    }
                }
            }
        }

        stage('Terraform Init') {
            steps {
                sh '''
                cd /var/lib/jenkins/workspace/eks_deployment/EKS_Terraform
                terraform init
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
