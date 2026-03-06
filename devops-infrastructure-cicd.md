---
layout: default
title: Infrastructure as Code
nav_order: 6
---

# DevOps: Infrastructure as Code & CI/CD Pipelines

## Terraform Project Structure & Remote Backend

**Flow:** Engineer writes HCL → commits to Git → CI runs `terraform plan` → reviewer approves → CI runs `terraform apply` → state is stored remotely in S3, locked via DynamoDB.

Without a remote backend, the state file (`terraform.tfstate`) lives on whoever's laptop ran `apply` last. A second engineer runs `apply` from their machine with stale state and overwrites infrastructure. DynamoDB locking prevents concurrent applies from corrupting state.

### Project Layout

```
infrastructure/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── ecs-service/
│       └── ...
└── backend.tf
```

### Remote Backend Configuration

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "acme-terraform-state"
    key            = "environments/dev/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
  }
}
```

### DynamoDB Lock Table (One-Time Setup)

```hcl
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### Initializing

```bash
cd infrastructure/environments/dev
terraform init -backend-config="key=environments/dev/terraform.tfstate"
```

> **Security Note:** The S3 state file contains every resource attribute in plaintext — including database passwords, private keys, and API tokens that Terraform manages. Enable S3 bucket versioning (to recover from corruption), enforce SSE-S3 or SSE-KMS encryption, and restrict bucket access to the CI service role only. Never commit `.tfstate` to Git.

---

## Jenkins & Terraform Apply

**Flow:** PR merged → Jenkins webhook fires → pipeline checks out code → `terraform plan -out=plan.bin` → artifact saved → manual approval gate → `terraform apply plan.bin` (the exact reviewed plan, not a re-computed one).

The saved plan file is critical. Without it, there's a gap between what the reviewer approved and what actually gets applied — another commit or a resource drift could change the plan between stages.

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

---

## Building & Deploying on Jenkins

**Flow:** Git push → webhook → Jenkins pipeline: **Checkout → Build → Test → Publish Image → Deploy**.

Each stage gates the next. If tests fail, the image never gets published. If the image fails to push, deploy never runs. This prevents half-baked artifacts from reaching any environment.

### Full Pipeline: PHP Service with Docker

```groovy
pipeline {
    agent { label 'docker' }

    environment {
        ECR_REPO    = '123456789012.dkr.ecr.eu-west-1.amazonaws.com/order-service'
        IMAGE_TAG   = "${env.GIT_COMMIT[0..7]}"
        COMPOSE_FILE = 'docker-compose.ci.yml'
    }

    stages {
        stage('Build') {
            steps {
                sh """
                    docker build \
                        --target production \
                        --build-arg COMPOSER_AUTH=\${COMPOSER_AUTH} \
                        -t ${ECR_REPO}:${IMAGE_TAG} .
                """
            }
        }

        stage('Test') {
            steps {
                sh """
                    docker compose -f ${COMPOSE_FILE} up -d db wiremock
                    docker compose -f ${COMPOSE_FILE} run --rm app \
                        php vendor/bin/phpunit --testsuite=unit
                    docker compose -f ${COMPOSE_FILE} run --rm app \
                        php vendor/bin/phpunit --testsuite=integration
                """
            }
            post {
                always {
                    sh "docker compose -f ${COMPOSE_FILE} down -v"
                    junit 'build/reports/**/*.xml'
                }
            }
        }

        stage('Push Image') {
            when { branch 'main' }
            steps {
                sh """
                    aws ecr get-login-password --region eu-west-1 \
                        | docker login --username AWS --password-stdin ${ECR_REPO}
                    docker push ${ECR_REPO}:${IMAGE_TAG}
                    docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REPO}:latest
                    docker push ${ECR_REPO}:latest
                """
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh """
                    aws ecs update-service \
                        --cluster production \
                        --service order-service \
                        --force-new-deployment \
                        --region eu-west-1
                """
            }
        }
    }
}
```

> **Security Note:** The `COMPOSER_AUTH` build arg (for private Packagist repos) is visible in `docker history`. Use multi-stage builds where the auth token is only present in the builder stage, not the final production image. Better yet, use Docker BuildKit secrets: `--secret id=composer_auth,src=./auth.json` with `RUN --mount=type=secret,id=composer_auth`.

---

## AWS SSM Parameter Store

**Flow:** Engineer stores secret in SSM → application reads it at boot (or Terraform references it during deploy) → no secrets in Git, no secrets in environment variable definitions committed to repos.

Hardcoding `DB_PASSWORD=hunter2` in a `.env` file that gets committed (even to a private repo) is a matter of time before it leaks. SSM Parameter Store provides encrypted, versioned, auditable secret storage.

### Storing Parameters (CLI)

```bash
# String parameter (non-sensitive config)
aws ssm put-parameter \
    --name "/order-service/prod/database/host" \
    --value "orders-db.cluster-abc123.eu-west-1.rds.amazonaws.com" \
    --type String \
    --region eu-west-1

# SecureString (encrypted with KMS)
aws ssm put-parameter \
    --name "/order-service/prod/database/password" \
    --value "s3cureP@ssw0rd!" \
    --type SecureString \
    --key-id "alias/order-service-key" \
    --region eu-west-1
```

### Referencing in Terraform (ECS Task Definition)

```hcl
resource "aws_ecs_task_definition" "order_service" {
  family                = "order-service"
  container_definitions = jsonencode([
    {
      name  = "app"
      image = "${var.ecr_repo}:${var.image_tag}"
      secrets = [
        {
          name      = "DB_HOST"
          valueFrom = "arn:aws:ssm:eu-west-1:123456789012:parameter/order-service/prod/database/host"
        },
        {
          name      = "DB_PASSWORD"
          valueFrom = "arn:aws:ssm:eu-west-1:123456789012:parameter/order-service/prod/database/password"
        }
      ]
    }
  ])

  execution_role_arn = aws_iam_role.ecs_execution.arn
}
```

### IAM Policy for ECS to Read Secrets

```hcl
resource "aws_iam_role_policy" "ecs_ssm_access" {
  name = "ecs-ssm-read"
  role = aws_iam_role.ecs_execution.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["ssm:GetParameters", "ssm:GetParameter"]
        Resource = "arn:aws:ssm:eu-west-1:123456789012:parameter/order-service/prod/*"
      },
      {
        Effect   = "Allow"
        Action   = ["kms:Decrypt"]
        Resource = "arn:aws:kms:eu-west-1:123456789012:key/key-id-here"
      }
    ]
  })
}
```

### Reading in PHP at Runtime (Fallback for Non-ECS)

```php
use Aws\Ssm\SsmClient;

class ParameterStoreConfig
{
    private SsmClient $ssm;

    public function __construct(string $region = 'eu-west-1')
    {
        $this->ssm = new SsmClient(['region' => $region, 'version' => 'latest']);
    }

    public function get(string $path): string
    {
        $result = $this->ssm->getParameter([
            'Name'           => $path,
            'WithDecryption' => true,
        ]);

        return $result['Parameter']['Value'];
    }

    /** @return array<string, string> */
    public function getByPath(string $prefix): array
    {
        $result = $this->ssm->getParametersByPath([
            'Path'           => $prefix,
            'WithDecryption' => true,
            'Recursive'      => true,
        ]);

        $params = [];
        foreach ($result['Parameters'] as $param) {
            $key = basename($param['Name']);
            $params[$key] = $param['Value'];
        }

        return $params;
    }
}
```

> **Security Note:** SSM `GetParameter` calls are logged in CloudTrail — you get a full audit trail of who accessed which secret and when. But: if your application logs the values it reads (`$logger->info('DB config', $params)`), you've just written the plaintext secret to your log aggregator. Treat SSM values as opaque — log the parameter *name*, never the *value*.

---

## GitHub CLI (`gh pr`)

**Flow:** Branch → commit → push → `gh pr create` → reviewer approves → `gh pr merge` — all without leaving the terminal.

Switching to the GitHub UI to create a PR, copy a link, assign reviewers, and merge breaks flow. `gh` keeps you in the terminal where your context already lives.

### Common Workflow

```bash
# Create a PR with title, body, and reviewer
gh pr create \
    --title "Add distributed lock to invoice generation" \
    --body "Prevents duplicate invoices when SQS delivers the same message to multiple pods." \
    --reviewer teammate-handle \
    --label "backend"

# List open PRs assigned to you
gh pr list --assignee @me

# View a specific PR (diff, checks, comments)
gh pr view 142
gh pr diff 142

# Check CI status
gh pr checks 142

# Approve and merge (squash)
gh pr review 142 --approve --body "LGTM"
gh pr merge 142 --squash --delete-branch

# Quick: create PR from current branch with editor
gh pr create --fill  # auto-fills title from branch, body from commits
```

### Checking Out a Teammate's PR Locally

```bash
gh pr checkout 142
# runs tests locally, reviews code, then:
gh pr review 142 --approve
```

### Linking PRs to Issues

```bash
gh pr create \
    --title "Fix race condition in vehicle upsert" \
    --body "Closes #87. Uses ON CONFLICT instead of check-then-insert."
```

GitHub auto-closes issue `#87` when the PR merges.

> **Reliability Note:** `gh pr merge --auto` enables auto-merge when all checks pass — useful for dependabot PRs. But it merges the moment checks go green, even if you pushed a "wait, one more fix" commit that hasn't been CI'd yet. Use `--auto` only for PRs where the branch is final. For active development branches, merge manually after confirming the latest commit passed.

---

## `npm install` vs. `npm ci`

**Flow (CI):** Checkout → `npm ci` → test → build → deploy. Never `npm install` in CI.

`npm install` resolves dependencies against `package.json` and *updates* `package-lock.json` if versions drift. In CI, this means your build might use different dependency versions than what the developer tested locally. `npm ci` enforces the lockfile as the single source of truth.

### Comparison

| Behavior                        | `npm install`                         | `npm ci`                              |
|---------------------------------|---------------------------------------|---------------------------------------|
| Reads                           | `package.json` (primary)              | `package-lock.json` (primary)         |
| Modifies `package-lock.json`    | Yes, if tree differs                  | Never — fails if lockfile is outdated |
| Deletes `node_modules/`         | No — merges incrementally             | Yes — clean install every time        |
| Speed (cold)                    | Slower (resolves tree)                | Faster (skips resolution)             |
| Deterministic                   | No                                    | Yes                                   |
| Use in                          | Local development                     | CI/CD pipelines                       |

### Jenkinsfile Stage

```groovy
stage('Install Dependencies') {
    steps {
        sh '''
            node --version
            npm --version
            npm ci --ignore-scripts  # skip postinstall in CI if not needed
        '''
    }
}
```

### When `npm ci` Fails

```bash
$ npm ci
npm ERR! `npm ci` can only install packages when your package.json
npm ERR! and package-lock.json are in sync.

# Fix: developer forgot to commit the updated lockfile
npm install          # regenerates lockfile
git add package-lock.json
git commit -m "Sync lockfile after dependency update"
```

### `.npmrc` for CI Environments

```ini
# .npmrc (committed to repo)
engine-strict=true
save-exact=true
audit-level=high
```

`save-exact=true` pins exact versions in `package.json` (no `^` or `~`), reducing lockfile churn and version surprises.

> **Reliability Note:** `npm ci` deletes and recreates `node_modules/` from scratch on every run. On large projects this takes 30–60 seconds. To speed up CI, cache `~/.npm` (npm's download cache, not `node_modules/`) between builds. Jenkins: use the `stash`/`unstash` steps or a shared volume. This cuts install time to ~5 seconds for warm caches while keeping the deterministic lockfile guarantee.
