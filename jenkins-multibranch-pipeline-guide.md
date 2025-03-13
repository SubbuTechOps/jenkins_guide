# Setting Up a Multi-Branch Jenkins Pipeline: A Practical Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Understanding Multi-Branch Pipelines](#understanding-multi-branch-pipelines)
4. [Step-by-Step Setup Guide](#step-by-step-setup-guide)
5. [Real-World Use Case: Microservice Development](#real-world-use-case-microservice-development)
6. [Jenkins Configuration as Code](#jenkins-configuration-as-code)
7. [Best Practices](#best-practices)
8. [Troubleshooting Common Issues](#troubleshooting-common-issues)
9. [Advanced Configuration Options](#advanced-configuration-options)
10. [References](#references)

## Introduction

Jenkins multi-branch pipelines automatically detect, manage, and execute pipelines for branches in your source code repository. This approach helps development teams implement CI/CD workflows across all branches, improving collaboration and code quality.

Key benefits include:
- **Automatic branch detection**: New branches are automatically discovered and pipeline jobs created
- **Isolated testing environments**: Each branch gets its own pipeline execution
- **Pull/Merge request integration**: Automatic testing of PRs/MRs before merging
- **Consistent workflows**: Enforce the same quality standards across all branches

## Prerequisites

Before setting up a multi-branch pipeline, ensure you have:

- Jenkins (version 2.0 or later) installed
- Jenkins administrator access
- Git, GitHub, Bitbucket, or GitLab plugin installed
- Appropriate source code repository credentials configured in Jenkins
- Jenkinsfile in your repository (or knowledge of how to create one)
- Required build tools installed on Jenkins agents

## Understanding Multi-Branch Pipelines

A multi-branch pipeline:
- Scans a repository for branches containing a Jenkinsfile
- Creates a pipeline job for each branch with a Jenkinsfile
- Automatically removes pipeline jobs when branches are deleted
- Supports declarative or scripted pipeline syntax
- Can prioritize specific branches (e.g., main/master)
- Enables branch-specific behaviors and configurations

## Step-by-Step Setup Guide

### 1. Install Required Plugins

Ensure these plugins are installed:
- Git/GitHub/Bitbucket/GitLab (depending on your repository)
- Pipeline
- Multibranch Pipeline
- Blue Ocean (recommended for improved UI)

Navigate to **Manage Jenkins > Plugins > Available** and install the required plugins.

### 2. Configure Credentials

1. Go to **Manage Jenkins > Credentials > System > Global credentials**
2. Click **Add Credentials**
3. Select the appropriate credential type (usually Username with password or SSH key)
4. Enter your repository access credentials
5. Provide an ID (e.g., "github-repo-access")
6. Click **OK**

### 3. Create a Multi-Branch Pipeline

1. From Jenkins dashboard, click **New Item**
2. Enter a name for your pipeline
3. Select **Multibranch Pipeline**
4. Click **OK**

### 4. Configure Branch Sources

1. In the configuration page, under **Branch Sources**, click **Add source**
2. Select your repository type (GitHub, Git, Bitbucket, etc.)
3. Enter the repository URL
4. Select the credentials created earlier
5. Configure **Behaviors** as needed
   - Discover branches
   - Discover pull requests
   - Specify clone/checkout options

### 5. Set Build Configuration

1. Under **Build Configuration**:
   - Set **Mode** to "by Jenkinsfile"
   - Set **Script Path** to the relative path of your Jenkinsfile (default: "Jenkinsfile")

### 6. Set Scan Repository Triggers

1. Under **Scan Multibranch Pipeline Triggers**:
   - Check **Periodically if not otherwise run**
   - Set interval (e.g., 1 hour)
   - Or configure webhooks for immediate scanning

### 7. Set Orphaned Item Strategy

1. Under **Orphaned Item Strategy**:
   - Select **Discard old items**
   - Set days to keep old items (e.g., 7 days)

### 8. Save and Scan

1. Click **Save**
2. Jenkins will automatically scan the repository
3. Pipeline jobs will be created for branches with a Jenkinsfile

## Real-World Use Case: Microservice Development

### Scenario
Consider a microservice architecture with multiple development teams working on different features simultaneously. Each team creates feature branches from the main branch.

### Requirements
- All code must pass unit tests
- Integration tests must pass before merging to main
- Code quality checks must be enforced
- Deployment to staging for main branch changes
- Deployment to production requires manual approval

### Jenkinsfile Example

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.8.4-openjdk-11
    command:
    - cat
    tty: true
  - name: docker
    image: docker:latest
    command:
    - cat
    tty: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
      type: Socket
"""
        }
    }
    
    environment {
        SERVICE_NAME = "user-service"
        DOCKER_REGISTRY = "your-registry.com"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                container('maven') {
                    sh 'mvn test'
                }
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Code Quality') {
            steps {
                container('maven') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        
        stage('Integration Tests') {
            when {
                anyOf {
                    branch 'main'
                    branch 'release/*'
                    changeRequest()
                }
            }
            steps {
                container('maven') {
                    sh 'mvn verify -DskipUnitTests'
                }
            }
            post {
                always {
                    junit 'target/failsafe-reports/*.xml'
                }
            }
        }
        
        stage('Build Docker Image') {
            when {
                anyOf {
                    branch 'main'
                    branch 'release/*'
                }
            }
            steps {
                container('docker') {
                    sh """
                        docker build -t ${DOCKER_REGISTRY}/${SERVICE_NAME}:${env.BRANCH_NAME}-${env.BUILD_NUMBER} .
                        docker tag ${DOCKER_REGISTRY}/${SERVICE_NAME}:${env.BRANCH_NAME}-${env.BUILD_NUMBER} ${DOCKER_REGISTRY}/${SERVICE_NAME}:latest
                    """
                }
            }
        }
        
        stage('Push Docker Image') {
            when {
                anyOf {
                    branch 'main'
                    branch 'release/*'
                }
            }
            steps {
                container('docker') {
                    withCredentials([string(credentialsId: 'docker-registry-token', variable: 'DOCKER_TOKEN')]) {
                        sh """
                            echo ${DOCKER_TOKEN} | docker login ${DOCKER_REGISTRY} -u jenkins --password-stdin
                            docker push ${DOCKER_REGISTRY}/${SERVICE_NAME}:${env.BRANCH_NAME}-${env.BUILD_NUMBER}
                            docker push ${DOCKER_REGISTRY}/${SERVICE_NAME}:latest
                        """
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                container('kubectl') {
                    withKubeConfig([credentialsId: 'kubernetes-staging']) {
                        sh 'kubectl apply -f kubernetes/staging -n staging'
                        sh 'kubectl set image deployment/${SERVICE_NAME} ${SERVICE_NAME}=${DOCKER_REGISTRY}/${SERVICE_NAME}:latest -n staging'
                    }
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'release/*'
            }
            steps {
                timeout(time: 24, unit: 'HOURS') {
                    input message: 'Approve deployment to production?', submitter: 'authorized-approvers'
                }
                container('kubectl') {
                    withKubeConfig([credentialsId: 'kubernetes-production']) {
                        sh 'kubectl apply -f kubernetes/production -n production'
                        sh 'kubectl set image deployment/${SERVICE_NAME} ${SERVICE_NAME}=${DOCKER_REGISTRY}/${SERVICE_NAME}:${env.BRANCH_NAME}-${env.BUILD_NUMBER} -n production'
                    }
                }
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            cleanWs()
        }
        success {
            slackSend channel: '#builds', color: 'good', message: "SUCCESS: Job '${env.JOB_NAME}' [${env.BUILD_NUMBER}] - Branch: ${env.BRANCH_NAME}"
        }
        failure {
            slackSend channel: '#builds', color: 'danger', message: "FAILED: Job '${env.JOB_NAME}' [${env.BUILD_NUMBER}] - Branch: ${env.BRANCH_NAME}"
        }
    }
}
```

### Team Workflow

1. Developer creates a feature branch from main
2. Jenkinsfile in the repository defines the pipeline steps
3. Jenkins automatically detects the new branch and creates a pipeline
4. Developer commits code and pushes to the feature branch
5. Jenkins pipeline runs: build, unit tests, code quality
6. Developer creates a pull request to merge to main
7. Jenkins runs integration tests on the PR branch
8. If all tests pass, the PR can be approved and merged
9. The main branch pipeline runs, building a Docker image and deploying to staging
10. For releases, a release branch is created from main
11. After testing in staging, the release can be approved for production deployment

## Jenkins Configuration as Code

For organizations managing many Jenkins instances or requiring consistent setups, you can define your multi-branch pipeline configuration using the Jenkins Configuration as Code (JCasC) plugin.

Example JCasC YAML for a multi-branch pipeline:

```yaml
jobs:
  - script: >
      multibranchPipelineJob('my-service') {
        branchSources {
          github {
            id('my-service-repo')
            scanCredentialsId('github-credentials')
            repoOwner('my-organization')
            repository('my-service')
            buildOriginBranch(true)
            buildOriginBranchWithPR(true)
            buildOriginPRMerge(true)
          }
        }
        orphanedItemStrategy {
          discardOldItems {
            numToKeep(7)
          }
        }
        triggers {
          periodicFolderTrigger {
            interval('1h')
          }
        }
        factory {
          workflowBranchProjectFactory {
            scriptPath('Jenkinsfile')
          }
        }
      }
```

## Best Practices

### Repository Organization
- Keep the Jenkinsfile in the root directory
- Use environment variables for configuration
- Consider separating infrastructure code (e.g., Kubernetes manifests)

### Pipeline Design
- Break the pipeline into logical stages
- Keep stages focused on a single responsibility
- Use when conditions to control execution based on branch
- Implement proper cleanup in post steps

### Performance Optimization
- Use parallel stages for independent tasks
- Implement caching for dependencies
- Use ephemeral agents to ensure clean environments

### Security
- Store credentials in Jenkins credentials store, not in the Jenkinsfile
- Use Jenkins' built-in secret handling
- Limit repository permissions to necessary operations
- Implement quality gates for security scanning

### Branching Strategy
- Define a clear branching strategy (e.g., GitFlow, GitHub Flow)
- Configure branch specific behaviors
- Implement branch protection rules in your repository

## Troubleshooting Common Issues

### Pipeline Not Detecting Branches
- Verify repository URL and credentials
- Check branch discovery behavior settings
- Ensure branches contain a Jenkinsfile at the specified path
- Check Jenkins logs for scanning errors

### Pipeline Execution Failures
- Examine the pipeline logs for error messages
- Verify agent configurations
- Check resource availability (e.g., disk space, memory)
- Validate environment-specific configurations

### Webhook Integration Issues
- Verify webhook URL and authentication
- Check network connectivity between repository and Jenkins
- Examine webhook delivery logs in your repository
- Test webhook manually

### Performance Problems
- Review pipeline design for optimization opportunities
- Check agent provisioning and utilization
- Monitor disk space and cleanup old builds
- Implement artifact caching

## Advanced Configuration Options

### Branch Indexing
Fine-tune how Jenkins discovers branches:

```groovy
branchSources {
  git {
    remote('https://github.com/organization/repository.git')
    credentialsId('github-credentials')
    traits {
      // Discover branches
      branchDiscoveryTrait {
        strategyId(3) // Detect all branches
      }
      // Discover PRs from origin
      pruneStaleBranchTrait()
      // Discover PRs from forks
      gitHubPullRequestDiscovery {
        strategyId(1) // PR merge with base branch
      }
      // Filter by name (with regular expression)
      headRegexFilter('^(main|develop|feature/.*|release/.*)$')
    }
  }
}
```

### Property Strategy
Apply different properties to different branches:

```groovy
properties {
  // Configure different properties for different branches
  folderPropertyStrategy {
    properties {
      named('main') {
        disableConcurrentBuilds()
        buildRetentionStrategy {
          buildDays(30)
        }
      }
      named('develop') {
        disableConcurrentBuilds()
        buildRetentionStrategy {
          buildDays(14)
        }
      }
      regexp(/feature\/.*/) {
        buildRetentionStrategy {
          buildDays(7)
        }
      }
    }
  }
}
```

### Build Strategies
Control which branches build automatically:

```groovy
buildStrategies {
  buildRegularBranches()
  buildChangeRequests {
    ignoreTargetOnlyChanges(true)
    ignoreUntrustedChanges(true)
  }
  skipInitialBuildOnFirstBranchIndexing()
  rateLimitBranchCreation {
    count(3)
    durationName('hour')
  }
}
```

## References

- [Jenkins User Documentation](https://www.jenkins.io/doc/)
- [Pipeline Syntax Reference](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Multibranch Pipeline Job](https://www.jenkins.io/doc/book/pipeline/multibranch/)
- [Jenkins Configuration as Code](https://www.jenkins.io/projects/jcasc/)
- [Pipeline Best Practices](https://www.jenkins.io/doc/book/pipeline/pipeline-best-practices/)
