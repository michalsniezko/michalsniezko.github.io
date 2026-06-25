---
layout: default
title: Jenkins Multibranch Pipelines
parent: Infrastructure & CI/CD
nav_order: 14
---

## Jenkins Multibranch Pipelines

A regular Jenkins pipeline job runs one branch. A multibranch pipeline job automatically discovers every branch in a repository and creates a separate job for each one. When a developer pushes a new branch, Jenkins finds it and starts running the pipeline. When the branch is deleted, Jenkins cleans up the job.

The pipeline definition lives in a `Jenkinsfile` at the root of the repository. The same file runs for every branch - conditional logic inside it determines what each branch actually does.

---

## How Branch Discovery Works

Jenkins periodically scans the repository (branch indexing) for new, changed, and deleted branches. When it finds a branch:

- If no job exists for it yet: Jenkins creates one and queues a build
- If the job already exists: Jenkins queues a build only if the branch HEAD has changed
- If the branch no longer exists in the repository: Jenkins marks the job as orphaned and removes it after a configured grace period

Branch indexing itself triggers a build - this is distinct from a real code push. The first thing most pipelines do is abort builds caused by indexing rather than commits:

```groovy
stage('Setup') {
    steps {
        script {
            // Indexing scans don't change HEAD - abort these to avoid
            // spurious builds showing up in the branch history
            if (currentBuild.buildCauses.toString().contains('BranchIndexingCause')) {
                currentBuild.result = 'ABORTED'
                error 'Build aborted - branch indexing, not a real trigger'
            }
        }
    }
}
```

Without this guard, every Jenkins rescan creates a new build entry for every branch - polluting the build history with no-op runs.

---

## A Minimal Jenkinsfile

```groovy
pipeline {
    agent { label 'docker' }

    options {
        buildDiscarder(logRotator(numToKeepStr: '30'))
        disableConcurrentBuilds()
        timeout(time: 1, unit: 'HOURS')
        timestamps()
        ansiColor('xterm')
    }

    stages {
        stage('Build') {
            steps {
                sh 'make build IMAGE_TAG=${GIT_COMMIT[0..7]}'
            }
        }

        stage('Test') {
            steps {
                sh 'make test'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'build/reports/**/*.xml'
                }
            }
        }

        stage('Push') {
            when { branch 'main' }
            steps {
                sh 'make push IMAGE_TAG=${GIT_COMMIT[0..7]}'
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh 'make deploy ENV=qa IMAGE_TAG=${GIT_COMMIT[0..7]}'
            }
        }
    }

    post {
        success {
            echo "Pipeline passed for ${env.GIT_BRANCH}"
        }
        failure {
            echo "Pipeline failed for ${env.GIT_BRANCH}"
        }
    }
}
```

Build and test run on every branch. Push and deploy run only on `main`. Feature branches get CI feedback without accidentally deploying.

---

## The options Block

```groovy
options {
    buildDiscarder(logRotator(numToKeepStr: '30'))
    disableConcurrentBuilds()
    timeout(time: 1, unit: 'HOURS')
    timestamps()
    ansiColor('xterm')
}
```

`buildDiscarder` keeps only the last 30 builds per branch. Without it, Jenkins retains every build indefinitely across potentially dozens of branches.

`disableConcurrentBuilds` allows only one running build per branch at a time. If a build is already running and a new commit arrives, the new build queues and waits. This prevents two deploys from racing on the same branch.

`timeout` kills the entire pipeline after 1 hour. Essential guard against hung Docker builds or stuck `aws ecs wait` calls that would otherwise block the executor indefinitely.

`ansiColor` renders ANSI color codes from build output (test frameworks, Docker) in the Jenkins console rather than showing raw escape sequences.

---

## Branch-Conditional Behavior

The `when` directive controls whether a stage runs based on the current branch:

```groovy
// Run only on main
stage('Deploy to production') {
    when { branch 'main' }
    steps { ... }
}

// Run on any branch matching a pattern
stage('Deploy preview') {
    when { branch pattern: 'feature/*', comparator: 'GLOB' }
    steps { ... }
}

// Run on all branches except main
stage('Notify PR reviewers') {
    when {
        not { branch 'main' }
    }
    steps { ... }
}

// Combine conditions
stage('Tag release') {
    when {
        allOf {
            branch 'main'
            environment name: 'RELEASE', value: 'true'
        }
    }
    steps { ... }
}
```

The `GIT_BRANCH` environment variable holds the full branch name as Jenkins sees it (e.g. `origin/main` or `main` depending on configuration). Use the `branch` directive in `when` rather than string-matching `GIT_BRANCH` directly - `branch` normalises the comparison.

---

## Parameters

Parameters in multibranch pipelines behave differently from regular jobs. The first run on a new branch has no parameters - Jenkins must run the pipeline once to discover them. This means:

- The first build after branching always runs with defaults
- Parameter changes take effect on the next build after Jenkins re-reads the Jenkinsfile

```groovy
parameters {
    choice(
        name: 'ENV',
        choices: ['qa', 'staging', 'production'],
        description: 'Environment to deploy to'
    )
    booleanParam(
        name: 'SKIP_TESTS',
        defaultValue: false,
        description: 'Skip test stage (use for hotfixes only)'
    )
    string(
        name: 'IMAGE_TAG',
        defaultValue: '',
        description: 'Override image tag (leave empty to use git SHA)'
    )
}
```

Inside the pipeline, parameters are read via `params.ENV`, `params.SKIP_TESTS`, etc. Provide sensible defaults - the first build will use them.

A common pattern is computing a value from git when no parameter is provided:

```groovy
script {
    env.DEPLOY_TAG = params.IMAGE_TAG?.trim() 
        ? params.IMAGE_TAG 
        : env.GIT_COMMIT.take(7)
}
```

---

## Parallel Stages

Long test suites can be split across parallel stages to reduce wall-clock time:

```groovy
stage('Tests') {
    parallel {
        stage('Unit tests') {
            steps {
                sh 'make test-unit'
            }
        }
        stage('Integration tests') {
            steps {
                sh 'make test-integration'
            }
        }
        stage('Lint') {
            steps {
                sh 'make lint'
            }
        }
    }
}
```

Parallel stages run on the same agent by default. If your tests spin up Docker containers with fixed port mappings, parallel stages will conflict. Either use randomised ports, separate networks per stage, or run each parallel stage on its own agent.

---

## Shared Libraries

Jenkinsfiles across many repositories often share the same boilerplate. Jenkins Shared Libraries extract reusable functions into a separate Git repository loaded at pipeline start:

```groovy
@Library('my-pipeline-library@v1.2') _

pipeline {
    agent { label 'docker' }

    stages {
        stage('Build') {
            steps {
                // function defined in the shared library
                buildAndPushImage(
                    image: 'my-service',
                    tag: env.GIT_COMMIT.take(7)
                )
            }
        }
    }
}
```

The library is loaded from a configured Git repository. The `@v1.2` pins the library to a tag - without pinning, all pipelines pull the latest version and can break simultaneously if the library changes.

The library source structure:

```
vars/
  buildAndPushImage.groovy   # callable as buildAndPushImage(...)
  deployToEcs.groovy
src/
  com/example/
    PipelineUtils.groovy     # Groovy class, imported explicitly
resources/
  scripts/
    smoke-test.sh            # static files, loaded with libraryResource()
```

A function in `vars/buildAndPushImage.groovy`:

```groovy
def call(Map args) {
    def image = args.image
    def tag   = args.tag

    sh "docker build -t ${image}:${tag} ."
    sh "docker push ${image}:${tag}"
}
```

Shared libraries are the correct place for: ECR login helpers, ECS deploy wrappers, Slack notification senders, and anything else that would be copy-pasted into ten Jenkinsfiles.

---

## Build Description

`currentBuild.description` sets the text shown next to the build number in the Jenkins UI. Useful for tracking which version was built or deployed without opening the full console log:

```groovy
script {
    def tag = env.GIT_COMMIT.take(7)
    currentBuild.description = "Tag: ${tag} | Branch: ${env.GIT_BRANCH}"
}
```

For deployments, include a link to the deployed environment:

```groovy
currentBuild.description = "<a href='https://qa.example.com' target='_blank'>QA</a> @ ${tag}"
```

HTML is rendered in the Jenkins build list.

---

## Orphaned Branch Strategy

When a branch is deleted from the repository, Jenkins marks its job as orphaned. Configure what happens to it:

In the multibranch pipeline job configuration:

```
Orphaned Item Strategy:
  Discard old items: checked
  Days to keep old items: 0
  Max # of old items to keep: 0
```

Setting both to 0 removes orphaned jobs immediately. With non-zero values, Jenkins keeps them for a grace period - useful if your team temporarily deletes and recreates branches.

---

## Common Pitfalls

### disableConcurrentBuilds can queue for a long time

If a deploy stage hangs (network timeout, stuck `aws ecs wait`), every subsequent push queues and waits. The pipeline-level `timeout` is the escape hatch - without it, the queue grows until someone manually aborts.

Set `timeout` at the pipeline level as a hard ceiling, and also at the stage level for known slow operations:

```groovy
stage('Wait for deployment') {
    options {
        timeout(time: 15, unit: 'MINUTES')
    }
    steps {
        sh 'aws ecs wait services-stable --cluster my-cluster --services my-service'
    }
}
```

### Branch names with slashes

GitHub branches like `feature/my-feature` create Jenkins job paths with slashes: `my-repo/feature/my-feature`. Some Jenkins configurations interpret this incorrectly. The multibranch job URL encodes slashes as `%2F`. If you reference branch names in shell commands or Terraform state paths, sanitise them:

```groovy
// Replace slashes and other unsafe chars with dashes, limit length
def safeBranchName = env.GIT_BRANCH
    .toLowerCase()
    .replaceAll('[^a-z0-9-]', '-')
    .take(50)
```

### First build on a new branch has no parameters

The first build always uses parameter defaults. If your pipeline deploys on every push and someone branches from main, the first build deploys with defaults - which may not be what they intended. Gate deploys behind explicit conditions:

```groovy
stage('Deploy') {
    when {
        allOf {
            branch 'main'
            not { equals expected: true, actual: params.SKIP_DEPLOY }
        }
    }
    steps { ... }
}
```

### Credentials in the Jenkinsfile

Never hardcode credentials. Use the `credentials()` binding or `withCredentials` block:

```groovy
environment {
    AWS_ACCESS_KEY_ID     = credentials('aws-deploy-key-id')
    AWS_SECRET_ACCESS_KEY = credentials('aws-deploy-key-secret')
}
```

Or for finer-grained scoping:

```groovy
withCredentials([
    string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN'),
    usernamePassword(
        credentialsId: 'registry-credentials',
        usernameVariable: 'REGISTRY_USER',
        passwordVariable: 'REGISTRY_PASS'
    )
]) {
    sh 'docker login -u $REGISTRY_USER -p $REGISTRY_PASS registry.example.com'
}
```

The credential values are masked in the console output. Passing them through `env` at pipeline scope makes them available in all stages but also visible to any stage that prints the environment.

---

### For AI agents

```
Jenkins multibranch pipeline: one Jenkinsfile in the repo root runs for every branch. Branch discovery via periodic scanning - guard against spurious indexing builds by checking buildCauses for BranchIndexingCause and aborting. Use when { branch 'main' } to gate push/deploy stages. options { disableConcurrentBuilds() } serialises builds per branch - always pair with timeout() or a hung build blocks the queue. Parameters are not available on the first build of a new branch - always set safe defaults. Sanitise branch names for use in URLs/state paths (slashes, special chars). Shared libraries loaded with @Library('name@version') _ extract reusable pipeline functions into vars/functionName.groovy. Never hardcode credentials - use credentials() binding or withCredentials block.
```

Reference: `https://michalsniezko.github.io/devops-infrastructure-cicd/jenkins-multibranch-pipelines.html`
