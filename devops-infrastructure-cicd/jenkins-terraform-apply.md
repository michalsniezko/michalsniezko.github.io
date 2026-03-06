---
layout: default
title: Jenkins & Terraform Apply
parent: Infrastructure as Code
nav_order: 2
---

## Jenkins & Terraform Apply

**Flow:** PR merged → Jenkins webhook fires → pipeline checks out code → `terraform plan -out=plan.bin` → artifact saved → manual approval gate → `terraform apply plan.bin` (the exact reviewed plan, not a re-computed one).

The saved plan file is critical. Without it, there's a gap between what the reviewer approved and what actually gets applied - another commit or a resource drift could change the plan between stages.

### Jenkinsfile

```groovy
pipeline {
    agent { label 'terraform' }

    environment {
        AWS_REGION       = 'eu-west-1'
        TF_IN_AUTOMATION = 'true'
        TF_DIR           = 'infrastructure/environments/prod'
    }

    stages {
        stage('Init') {
            steps {
                dir("${TF_DIR}") {
                    sh 'terraform init -input=false -no-color'
                }
            }
        }

        stage('Plan') {
            steps {
                dir("${TF_DIR}") {
                    sh 'terraform plan -input=false -no-color -out=tfplan.bin'
                    sh 'terraform show -no-color tfplan.bin > tfplan.txt'
                }
                // Archive both: binary for apply, text for review
                archiveArtifacts artifacts: "${TF_DIR}/tfplan.bin, ${TF_DIR}/tfplan.txt"
            }
        }

        stage('Approve') {
            when { branch 'main' }
            steps {
                // Human reviews tfplan.txt in Jenkins UI before proceeding
                input message: 'Review the plan artifact. Apply to production?',
                      ok: 'Apply'
            }
        }

        stage('Apply') {
            when { branch 'main' }
            steps {
                dir("${TF_DIR}") {
                    // Apply the EXACT plan that was reviewed
                    sh 'terraform apply -input=false -no-color tfplan.bin'
                }
            }
        }
    }

    post {
        failure {
            slackSend channel: '#infra-alerts',
                      message: "Terraform pipeline failed: ${env.BUILD_URL}"
        }
        always {
            cleanWs()
        }
    }
}
```

> **Reliability Note:** If Jenkins restarts between the Plan and Apply stages, the workspace (and the plan file) is gone. The `archiveArtifacts` step saves the plan to Jenkins' artifact storage so it survives restarts. For extra safety, upload `tfplan.bin` to S3 with the build number as key, and download it in the Apply stage rather than relying on workspace persistence.
