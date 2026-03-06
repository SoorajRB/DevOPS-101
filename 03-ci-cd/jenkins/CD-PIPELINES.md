# Continuous Deployment (CD) with Jenkins

## Table of Contents

1. [Introduction to Continuous Deployment](#1-introduction-to-continuous-deployment)
2. [Writing CD Pipelines in Jenkins](#2-writing-cd-pipelines-in-jenkins)
3. [VM Deployment - Deep Dive](#3-vm-deployment---deep-dive)
4. [Kubernetes Deployment - Deep Dive](#4-kubernetes-deployment---deep-dive)
5. [Helm Deployment - Deep Dive](#5-helm-deployment---deep-dive)
6. [Environment Management](#6-environment-management)
7. [Monitoring Deployments](#7-monitoring-deployments)
8. [Practice Exercises](#8-practice-exercises)

---

## 1. Introduction to Continuous Deployment

### What is CD and How It Differs from CI

**Continuous Integration (CI)** is the practice of automatically building and testing code every time a developer pushes changes. CI answers the question: *"Does my code compile and pass tests?"*

**Continuous Deployment (CD)** takes CI a step further. After code is built and tested, CD automatically deploys that code to one or more environments (staging, production, etc.). CD answers the question: *"How do I reliably get my tested code running in front of users?"*

There is an important distinction between two related terms:

- **Continuous Delivery**: Code is automatically built, tested, and prepared for release, but a **manual approval** is required before deploying to production.
- **Continuous Deployment**: Code is automatically built, tested, **and deployed to production** without any manual intervention.

In practice, most teams use Continuous Delivery (with a manual gate before production) rather than fully automated Continuous Deployment. This guide covers both approaches.

```
CI/CD Pipeline Flow:

  Code Push --> Build --> Unit Tests --> Integration Tests --> Deploy to Staging --> Approval Gate --> Deploy to Production
  |<------------ CI ------------->|     |<---------------------- CD ------------------------------>|
```

### Deployment Strategies

#### Rolling Deployment

A rolling deployment gradually replaces instances of the old version with the new version, one at a time (or in small batches). At any point during the deployment, some instances run the old version and some run the new version.

```
Time 0:  [v1] [v1] [v1] [v1]    (all instances on v1)
Time 1:  [v2] [v1] [v1] [v1]    (1 instance updated)
Time 2:  [v2] [v2] [v1] [v1]    (2 instances updated)
Time 3:  [v2] [v2] [v2] [v1]    (3 instances updated)
Time 4:  [v2] [v2] [v2] [v2]    (all instances on v2)
```

**Pros**: Zero downtime, minimal extra resources needed, easy to implement.
**Cons**: During rollout, two versions coexist (may cause compatibility issues). Rollback requires another rolling update.

#### Blue-Green Deployment

Two identical environments are maintained: "Blue" (current production) and "Green" (new version). Traffic is switched from Blue to Green all at once after the Green environment is verified.

```
Before switch:
  Load Balancer --> [Blue - v1] (LIVE)
                    [Green - v2] (IDLE, being tested)

After switch:
  Load Balancer --> [Green - v2] (LIVE)
                    [Blue - v1] (IDLE, available for rollback)
```

**Pros**: Instant rollback (switch traffic back to Blue), zero downtime, full testing of the new version before go-live.
**Cons**: Requires double the infrastructure, database migrations need careful handling.

#### Canary Deployment

A small percentage of traffic is routed to the new version first. If the new version performs well (no errors, good latency), traffic is gradually shifted until 100% goes to the new version.

```
Phase 1:  [v1] receives 95% traffic   |   [v2] receives 5% traffic
Phase 2:  [v1] receives 75% traffic   |   [v2] receives 25% traffic
Phase 3:  [v1] receives 50% traffic   |   [v2] receives 50% traffic
Phase 4:  [v1] receives 0% traffic    |   [v2] receives 100% traffic
```

**Pros**: Minimal risk (only a small percentage of users affected if something goes wrong), allows real-world validation.
**Cons**: More complex to set up, requires traffic-splitting infrastructure, monitoring is critical.

### Environments

A typical deployment pipeline moves code through multiple environments before reaching production:

| Environment | Purpose | Who Uses It | Deployment |
|-------------|---------|-------------|------------|
| **Development (Dev)** | Active development and early testing | Developers | Automatic on every push |
| **Staging** | Pre-production testing, mirrors production | QA team, Product owners | Automatic after CI passes |
| **Production** | Live environment serving real users | End users | Manual approval or automatic |

Some organizations add additional environments:

- **QA/Test**: Dedicated environment for quality assurance testing.
- **UAT (User Acceptance Testing)**: For business stakeholders to validate features.
- **Performance/Load**: Dedicated to performance and load testing.

### Approval Gates and Manual vs Automatic Deployment

**Automatic deployment** means code flows from one stage to the next without human intervention. This is common for development and staging environments.

**Manual approval gates** require a human to review and approve before the pipeline proceeds. This is the standard practice for production deployments in most organizations.

In Jenkins, approval gates are implemented using the `input` step:

```groovy
// Example of a manual approval gate in Jenkins
stage('Approve Production Deploy') {
    steps {
        input message: 'Deploy to production?',
              ok: 'Deploy',
              submitter: 'admin,release-team'
    }
}
```

Key considerations for approval gates:
- **Who can approve**: Limit approvers to authorized personnel (team leads, release managers).
- **Timeout**: Set a timeout so the pipeline does not hang indefinitely.
- **Information**: Provide the approver with enough context (build number, changelog, test results) to make an informed decision.

---

## 2. Writing CD Pipelines in Jenkins

### Full CD Jenkinsfile for VM Deployment

This Jenkinsfile handles the complete lifecycle: checkout, build, test, push to registry, and deploy to a VM via SSH. It includes staging and production stages with an approval gate.

```groovy
// =============================================================================
// CD PIPELINE: VM DEPLOYMENT
// =============================================================================
// This Jenkinsfile deploys a Dockerized Python Flask application to virtual
// machines via SSH. It builds the Docker image, pushes it to a registry, then
// SSHs into the target VM to pull and run the new image.
//
// Prerequisites:
//   - Jenkins credentials 'docker-registry-creds' (username/password for Docker Hub)
//   - Jenkins credentials 'vm-ssh-key' (SSH private key for the deploy user on VMs)
//   - Docker installed on Jenkins agent
//   - Docker installed on target VMs
//   - SSH access from Jenkins to target VMs
// =============================================================================

pipeline {

    // -------------------------------------------------------------------------
    // AGENT: Run this pipeline on any available Jenkins agent that has Docker.
    // In production, you would label agents: agent { label 'docker' }
    // -------------------------------------------------------------------------
    agent any

    // -------------------------------------------------------------------------
    // ENVIRONMENT: Define variables available to all stages.
    // These control image names, tags, registry, and target VM addresses.
    // -------------------------------------------------------------------------
    environment {
        // Docker image name (without registry prefix)
        APP_NAME = 'flask-app'

        // Docker registry URL. For Docker Hub, use 'docker.io'.
        // For a private registry, use something like 'registry.example.com:5000'.
        DOCKER_REGISTRY = 'docker.io'

        // Docker Hub organization or username
        DOCKER_ORG = 'mycompany'

        // Full image name including registry and org
        DOCKER_IMAGE = "${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}"

        // Tag the image with the Git commit SHA (short form) for traceability.
        // Every image can be traced back to exactly which commit produced it.
        IMAGE_TAG = "${env.GIT_COMMIT?.take(7) ?: env.BUILD_NUMBER}"

        // Jenkins credential IDs (these must be configured in Jenkins beforehand)
        DOCKER_CREDENTIALS_ID = 'docker-registry-creds'
        SSH_CREDENTIALS_ID = 'vm-ssh-key'

        // Target VM IP addresses or hostnames
        STAGING_VM = '192.168.1.100'
        PRODUCTION_VM = '192.168.1.200'

        // The user on the target VM that will run deployment commands
        DEPLOY_USER = 'deploy'

        // Port the application listens on inside the container
        APP_PORT = '5000'

        // Port exposed on the host VM
        HOST_PORT = '80'
    }

    // -------------------------------------------------------------------------
    // OPTIONS: Pipeline-level settings.
    // -------------------------------------------------------------------------
    options {
        // If the pipeline runs for more than 30 minutes, abort it.
        // This prevents stuck pipelines from consuming resources.
        timeout(time: 30, unit: 'MINUTES')

        // Keep only the last 10 builds to save disk space on the Jenkins server.
        buildDiscarder(logRotator(numToKeepStr: '10'))

        // Print timestamps next to each line of console output.
        // This is invaluable for debugging slow stages.
        timestamps()

        // If the pipeline fails, do not allow the same pipeline to
        // run again immediately (avoids rapid repeated failures).
        disableConcurrentBuilds()
    }

    // -------------------------------------------------------------------------
    // STAGES: The main body of the pipeline, executed in order.
    // -------------------------------------------------------------------------
    stages {

        // =====================================================================
        // STAGE 1: CHECKOUT
        // =====================================================================
        // Pull the source code from the repository. Jenkins does this
        // automatically for Multibranch Pipelines, but we make it explicit.
        // =====================================================================
        stage('Checkout') {
            steps {
                // 'checkout scm' uses the SCM configuration defined in the
                // Jenkins job (e.g., the Git repo URL and branch).
                checkout scm

                // Print the current commit for the build log
                sh 'echo "Building commit: $(git rev-parse --short HEAD)"'
            }
        }

        // =====================================================================
        // STAGE 2: BUILD DOCKER IMAGE
        // =====================================================================
        // Build the Docker image from the Dockerfile in the repository root.
        // We tag it with both the commit SHA and 'latest'.
        // =====================================================================
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image: ${DOCKER_IMAGE}:${IMAGE_TAG}"

                    // Build the Docker image. The '.' at the end is the build
                    // context (current directory, which contains the Dockerfile).
                    // --no-cache ensures a clean build (optional, remove for speed).
                    sh """
                        docker build \
                            -t ${DOCKER_IMAGE}:${IMAGE_TAG} \
                            -t ${DOCKER_IMAGE}:latest \
                            --label "git-commit=${env.GIT_COMMIT}" \
                            --label "build-number=${env.BUILD_NUMBER}" \
                            .
                    """
                }
            }
        }

        // =====================================================================
        // STAGE 3: RUN TESTS
        // =====================================================================
        // Run the test suite inside a container built from the image we just
        // created. This validates the built artifact, not just the source code.
        // =====================================================================
        stage('Run Tests') {
            steps {
                script {
                    echo 'Running tests inside Docker container...'

                    // Run the test suite inside the container.
                    // --rm: Remove the container after it exits.
                    // The 'pytest' command runs Python tests. Adjust for your stack.
                    sh """
                        docker run --rm \
                            ${DOCKER_IMAGE}:${IMAGE_TAG} \
                            python -m pytest tests/ -v --tb=short
                    """
                }
            }
        }

        // =====================================================================
        // STAGE 4: PUSH TO DOCKER REGISTRY
        // =====================================================================
        // Push the built and tested image to the Docker registry so that
        // target VMs (and any other environment) can pull it.
        // =====================================================================
        stage('Push to Registry') {
            steps {
                script {
                    echo "Pushing image to ${DOCKER_REGISTRY}..."

                    // Use Jenkins credentials to log in to the Docker registry.
                    // 'withCredentials' injects the username and password securely.
                    withCredentials([
                        usernamePassword(
                            credentialsId: DOCKER_CREDENTIALS_ID,
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        sh """
                            # Log in to Docker registry
                            echo "\${DOCKER_PASS}" | docker login ${DOCKER_REGISTRY} \
                                -u "\${DOCKER_USER}" --password-stdin

                            # Push both tags (commit SHA and latest)
                            docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                            docker push ${DOCKER_IMAGE}:latest

                            # Log out for security
                            docker logout ${DOCKER_REGISTRY}
                        """
                    }
                }
            }
        }

        // =====================================================================
        // STAGE 5: DEPLOY TO STAGING
        // =====================================================================
        // SSH into the staging VM, pull the new image, stop the old container,
        // and start a new container. This happens automatically (no approval).
        // =====================================================================
        stage('Deploy to Staging') {
            steps {
                script {
                    echo "Deploying ${DOCKER_IMAGE}:${IMAGE_TAG} to Staging (${STAGING_VM})..."

                    // sshagent injects the SSH private key into the agent so
                    // that 'ssh' and 'scp' commands can authenticate without
                    // prompting for a password.
                    sshagent(credentials: [SSH_CREDENTIALS_ID]) {
                        sh """
                            # SSH into the staging VM and execute deployment commands.
                            # -o StrictHostKeyChecking=no: Accept the host key automatically.
                            #   (In production, pre-populate known_hosts instead.)
                            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${STAGING_VM} << 'ENDSSH'

                                echo '=== Pulling new image ==='
                                docker pull ${DOCKER_IMAGE}:${IMAGE_TAG}

                                echo '=== Stopping old container (if running) ==='
                                docker stop ${APP_NAME} 2>/dev/null || true
                                docker rm ${APP_NAME} 2>/dev/null || true

                                echo '=== Starting new container ==='
                                docker run -d \\
                                    --name ${APP_NAME} \\
                                    --restart unless-stopped \\
                                    -p ${HOST_PORT}:${APP_PORT} \\
                                    -e FLASK_ENV=staging \\
                                    ${DOCKER_IMAGE}:${IMAGE_TAG}

                                echo '=== Waiting for container to be healthy ==='
                                sleep 10

                                echo '=== Health check ==='
                                curl -f http://localhost:${HOST_PORT}/health || exit 1

                                echo '=== Deployment to staging successful ==='
ENDSSH
                        """
                    }
                }
            }
        }

        // =====================================================================
        // STAGE 6: STAGING SMOKE TESTS
        // =====================================================================
        // Run quick smoke tests against the staging environment to verify the
        // deployment worked correctly before proceeding to production.
        // =====================================================================
        stage('Staging Smoke Tests') {
            steps {
                script {
                    echo 'Running smoke tests against staging...'
                    sh """
                        # Test that the application responds with HTTP 200
                        STATUS=\$(curl -s -o /dev/null -w '%{http_code}' \
                            http://${STAGING_VM}:${HOST_PORT}/health)

                        if [ "\${STATUS}" != "200" ]; then
                            echo "ERROR: Staging health check failed with status \${STATUS}"
                            exit 1
                        fi

                        echo "Staging smoke tests passed (HTTP \${STATUS})"
                    """
                }
            }
        }

        // =====================================================================
        // STAGE 7: APPROVAL GATE FOR PRODUCTION
        // =====================================================================
        // Pause the pipeline and wait for a human to approve the production
        // deployment. This is the Continuous Delivery model.
        // =====================================================================
        stage('Approve Production Deploy') {
            steps {
                // The 'input' step pauses the pipeline and shows a prompt
                // in the Jenkins UI. The pipeline will not proceed until
                // someone clicks 'Deploy' or the timeout expires.
                timeout(time: 24, unit: 'HOURS') {
                    input(
                        message: "Deploy ${DOCKER_IMAGE}:${IMAGE_TAG} to production?",
                        ok: 'Deploy to Production',
                        // Only these users/groups can approve:
                        submitter: 'admin,release-managers',
                        parameters: [
                            string(
                                name: 'DEPLOY_NOTES',
                                defaultValue: '',
                                description: 'Optional notes for this deployment'
                            )
                        ]
                    )
                }
            }
        }

        // =====================================================================
        // STAGE 8: DEPLOY TO PRODUCTION
        // =====================================================================
        // Same process as staging, but targets the production VM.
        // This stage only runs after manual approval.
        // =====================================================================
        stage('Deploy to Production') {
            steps {
                script {
                    echo "Deploying ${DOCKER_IMAGE}:${IMAGE_TAG} to Production (${PRODUCTION_VM})..."

                    sshagent(credentials: [SSH_CREDENTIALS_ID]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${PRODUCTION_VM} << 'ENDSSH'

                                echo '=== Pulling new image ==='
                                docker pull ${DOCKER_IMAGE}:${IMAGE_TAG}

                                echo '=== Saving current image tag for rollback ==='
                                # Record the currently running image tag so we can
                                # roll back if the new deployment fails.
                                CURRENT_IMAGE=\$(docker inspect --format='{{.Config.Image}}' ${APP_NAME} 2>/dev/null || echo "none")
                                echo "\${CURRENT_IMAGE}" > /tmp/${APP_NAME}_previous_image.txt

                                echo '=== Stopping old container ==='
                                docker stop ${APP_NAME} 2>/dev/null || true
                                docker rm ${APP_NAME} 2>/dev/null || true

                                echo '=== Starting new container ==='
                                docker run -d \\
                                    --name ${APP_NAME} \\
                                    --restart unless-stopped \\
                                    -p ${HOST_PORT}:${APP_PORT} \\
                                    -e FLASK_ENV=production \\
                                    ${DOCKER_IMAGE}:${IMAGE_TAG}

                                echo '=== Waiting for container to start ==='
                                sleep 15

                                echo '=== Health check ==='
                                if ! curl -f http://localhost:${HOST_PORT}/health; then
                                    echo 'ERROR: Health check failed! Rolling back...'
                                    docker stop ${APP_NAME} 2>/dev/null || true
                                    docker rm ${APP_NAME} 2>/dev/null || true

                                    PREV_IMAGE=\$(cat /tmp/${APP_NAME}_previous_image.txt)
                                    if [ "\${PREV_IMAGE}" != "none" ]; then
                                        docker run -d \\
                                            --name ${APP_NAME} \\
                                            --restart unless-stopped \\
                                            -p ${HOST_PORT}:${APP_PORT} \\
                                            -e FLASK_ENV=production \\
                                            \${PREV_IMAGE}
                                        echo "Rolled back to \${PREV_IMAGE}"
                                    fi
                                    exit 1
                                fi

                                echo '=== Production deployment successful ==='
ENDSSH
                        """
                    }
                }
            }
        }

        // =====================================================================
        // STAGE 9: PRODUCTION SMOKE TESTS
        // =====================================================================
        stage('Production Smoke Tests') {
            steps {
                script {
                    echo 'Running smoke tests against production...'
                    sh """
                        STATUS=\$(curl -s -o /dev/null -w '%{http_code}' \
                            http://${PRODUCTION_VM}:${HOST_PORT}/health)

                        if [ "\${STATUS}" != "200" ]; then
                            echo "ERROR: Production health check failed with status \${STATUS}"
                            exit 1
                        fi

                        echo "Production smoke tests passed (HTTP \${STATUS})"
                    """
                }
            }
        }
    }

    // -------------------------------------------------------------------------
    // POST: Actions that run after the pipeline completes (success or failure).
    // -------------------------------------------------------------------------
    post {

        // Run on every completion, regardless of result
        always {
            echo "Pipeline finished with status: ${currentBuild.currentResult}"

            // Clean up Docker images from the Jenkins agent to save disk space.
            sh """
                docker rmi ${DOCKER_IMAGE}:${IMAGE_TAG} 2>/dev/null || true
                docker rmi ${DOCKER_IMAGE}:latest 2>/dev/null || true
            """

            // Clean up the workspace
            cleanWs()
        }

        // Run only on success
        success {
            echo 'Deployment completed successfully!'

            // Send a Slack notification (requires Slack Notification Plugin).
            // Uncomment the lines below and configure the Slack plugin in Jenkins.
            // slackSend(
            //     channel: '#deployments',
            //     color: 'good',
            //     message: "SUCCESS: ${APP_NAME}:${IMAGE_TAG} deployed to production. " +
            //              "Build: ${env.BUILD_URL}"
            // )
        }

        // Run only on failure
        failure {
            echo 'Deployment FAILED!'

            // slackSend(
            //     channel: '#deployments',
            //     color: 'danger',
            //     message: "FAILURE: ${APP_NAME}:${IMAGE_TAG} deployment failed! " +
            //              "Build: ${env.BUILD_URL}"
            // )

            // Send email notification
            // mail(
            //     to: 'team@example.com',
            //     subject: "FAILED: ${APP_NAME} Deployment - Build #${env.BUILD_NUMBER}",
            //     body: """
            //         The deployment pipeline has failed.
            //         Build URL: ${env.BUILD_URL}
            //         Image: ${DOCKER_IMAGE}:${IMAGE_TAG}
            //         Please investigate immediately.
            //     """
            // )
        }
    }
}
```

### Full CD Jenkinsfile for Kubernetes/Helm Deployment

This Jenkinsfile deploys to Kubernetes using Helm charts. It includes image building, pushing to a registry, Helm-based deployment with rollback on failure, and staging/production stages.

```groovy
// =============================================================================
// CD PIPELINE: KUBERNETES / HELM DEPLOYMENT
// =============================================================================
// This Jenkinsfile deploys a Dockerized application to Kubernetes clusters
// using Helm charts. It supports staging and production environments with
// separate namespaces and Helm value files.
//
// Prerequisites:
//   - Jenkins credentials 'docker-registry-creds' (username/password)
//   - Jenkins credentials 'kubeconfig-staging' (kubeconfig file for staging cluster)
//   - Jenkins credentials 'kubeconfig-production' (kubeconfig file for prod cluster)
//   - Docker installed on Jenkins agent
//   - Helm 3.x installed on Jenkins agent
//   - kubectl installed on Jenkins agent
//   - Helm chart located in ./helm/flask-app/ directory in the repo
// =============================================================================

pipeline {

    // -------------------------------------------------------------------------
    // AGENT: Run on agents labeled 'kubernetes' that have kubectl and helm.
    // -------------------------------------------------------------------------
    agent {
        label 'kubernetes'
    }

    // -------------------------------------------------------------------------
    // ENVIRONMENT: Variables for the entire pipeline.
    // -------------------------------------------------------------------------
    environment {
        APP_NAME = 'flask-app'
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_ORG = 'mycompany'
        DOCKER_IMAGE = "${DOCKER_REGISTRY}/${DOCKER_ORG}/${APP_NAME}"
        IMAGE_TAG = "${env.GIT_COMMIT?.take(7) ?: env.BUILD_NUMBER}"

        // Credential IDs configured in Jenkins
        DOCKER_CREDENTIALS_ID = 'docker-registry-creds'
        KUBECONFIG_STAGING_ID = 'kubeconfig-staging'
        KUBECONFIG_PRODUCTION_ID = 'kubeconfig-production'

        // Helm chart path (relative to workspace)
        HELM_CHART_PATH = './helm/flask-app'

        // Helm release name (what Helm calls this deployment)
        HELM_RELEASE_NAME = 'flask-app'

        // Kubernetes namespaces for each environment
        STAGING_NAMESPACE = 'staging'
        PRODUCTION_NAMESPACE = 'production'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        // =====================================================================
        // STAGE 1: CHECKOUT
        // =====================================================================
        stage('Checkout') {
            steps {
                checkout scm
                sh 'echo "Building commit: $(git rev-parse --short HEAD)"'
            }
        }

        // =====================================================================
        // STAGE 2: BUILD DOCKER IMAGE
        // =====================================================================
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image: ${DOCKER_IMAGE}:${IMAGE_TAG}"
                    sh """
                        docker build \
                            -t ${DOCKER_IMAGE}:${IMAGE_TAG} \
                            -t ${DOCKER_IMAGE}:latest \
                            .
                    """
                }
            }
        }

        // =====================================================================
        // STAGE 3: RUN TESTS
        // =====================================================================
        stage('Run Tests') {
            steps {
                sh """
                    docker run --rm ${DOCKER_IMAGE}:${IMAGE_TAG} \
                        python -m pytest tests/ -v --tb=short
                """
            }
        }

        // =====================================================================
        // STAGE 4: PUSH TO DOCKER REGISTRY
        // =====================================================================
        stage('Push to Registry') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: DOCKER_CREDENTIALS_ID,
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        sh """
                            echo "\${DOCKER_PASS}" | docker login ${DOCKER_REGISTRY} \
                                -u "\${DOCKER_USER}" --password-stdin
                            docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                            docker push ${DOCKER_IMAGE}:latest
                            docker logout ${DOCKER_REGISTRY}
                        """
                    }
                }
            }
        }

        // =====================================================================
        // STAGE 5: LINT HELM CHART
        // =====================================================================
        // Validate the Helm chart before deploying. This catches syntax errors
        // and misconfigurations in templates.
        // =====================================================================
        stage('Lint Helm Chart') {
            steps {
                sh """
                    echo '=== Linting Helm chart ==='
                    helm lint ${HELM_CHART_PATH}

                    echo '=== Rendering templates (dry run) ==='
                    helm template ${HELM_RELEASE_NAME} ${HELM_CHART_PATH} \
                        --set image.repository=${DOCKER_IMAGE} \
                        --set image.tag=${IMAGE_TAG}
                """
            }
        }

        // =====================================================================
        // STAGE 6: DEPLOY TO STAGING (KUBERNETES)
        // =====================================================================
        // Deploy to the staging Kubernetes cluster using Helm.
        // Uses a staging-specific values file for environment configuration.
        // =====================================================================
        stage('Deploy to Staging') {
            steps {
                script {
                    echo "Deploying to Staging Kubernetes cluster..."

                    // Inject the kubeconfig file for the staging cluster.
                    // This file tells kubectl/helm how to connect to the cluster.
                    withCredentials([
                        file(credentialsId: KUBECONFIG_STAGING_ID, variable: 'KUBECONFIG')
                    ]) {
                        sh """
                            # Verify cluster connectivity
                            kubectl get nodes

                            # Create namespace if it doesn't exist
                            kubectl create namespace ${STAGING_NAMESPACE} \
                                --dry-run=client -o yaml | kubectl apply -f -

                            # Deploy or upgrade the Helm release.
                            # --install: Install the release if it doesn't exist.
                            # --wait: Wait for all pods to be ready before marking success.
                            # --timeout: Fail if pods are not ready within this time.
                            # -f: Use the staging values file for environment-specific config.
                            # --set: Override the image tag with the one we just built.
                            helm upgrade --install ${HELM_RELEASE_NAME} ${HELM_CHART_PATH} \
                                --namespace ${STAGING_NAMESPACE} \
                                -f ${HELM_CHART_PATH}/values-staging.yaml \
                                --set image.repository=${DOCKER_IMAGE} \
                                --set image.tag=${IMAGE_TAG} \
                                --wait \
                                --timeout 5m

                            # Verify the deployment rolled out successfully
                            kubectl rollout status deployment/${HELM_RELEASE_NAME} \
                                -n ${STAGING_NAMESPACE} \
                                --timeout=120s

                            # Show the current state of pods for the log
                            kubectl get pods -n ${STAGING_NAMESPACE} -l app.kubernetes.io/name=${APP_NAME}
                        """
                    }
                }
            }
        }

        // =====================================================================
        // STAGE 7: STAGING SMOKE TESTS
        // =====================================================================
        stage('Staging Smoke Tests') {
            steps {
                script {
                    withCredentials([
                        file(credentialsId: KUBECONFIG_STAGING_ID, variable: 'KUBECONFIG')
                    ]) {
                        sh """
                            # Get the Service ClusterIP and port
                            SVC_IP=\$(kubectl get svc ${HELM_RELEASE_NAME} \
                                -n ${STAGING_NAMESPACE} \
                                -o jsonpath='{.spec.clusterIP}')
                            SVC_PORT=\$(kubectl get svc ${HELM_RELEASE_NAME} \
                                -n ${STAGING_NAMESPACE} \
                                -o jsonpath='{.spec.ports[0].port}')

                            # Run a curl pod inside the cluster to test the service
                            kubectl run smoke-test --rm -i --restart=Never \
                                -n ${STAGING_NAMESPACE} \
                                --image=curlimages/curl:latest \
                                -- curl -sf http://\${SVC_IP}:\${SVC_PORT}/health

                            echo "Staging smoke tests passed!"
                        """
                    }
                }
            }
        }

        // =====================================================================
        // STAGE 8: APPROVAL GATE
        // =====================================================================
        stage('Approve Production Deploy') {
            steps {
                timeout(time: 24, unit: 'HOURS') {
                    input(
                        message: "Deploy ${APP_NAME}:${IMAGE_TAG} to production?",
                        ok: 'Deploy to Production',
                        submitter: 'admin,release-managers'
                    )
                }
            }
        }

        // =====================================================================
        // STAGE 9: DEPLOY TO PRODUCTION (KUBERNETES)
        // =====================================================================
        // Deploy to the production Kubernetes cluster. If the deployment fails,
        // automatically roll back to the previous Helm release.
        // =====================================================================
        stage('Deploy to Production') {
            steps {
                script {
                    echo "Deploying to Production Kubernetes cluster..."

                    withCredentials([
                        file(credentialsId: KUBECONFIG_PRODUCTION_ID, variable: 'KUBECONFIG')
                    ]) {
                        // Record the current Helm revision so we can roll back to it
                        // if the deployment fails.
                        def previousRevision = sh(
                            script: """
                                helm history ${HELM_RELEASE_NAME} \
                                    -n ${PRODUCTION_NAMESPACE} \
                                    --max 1 -o json 2>/dev/null \
                                    | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['revision'])" \
                                    2>/dev/null || echo '0'
                            """,
                            returnStdout: true
                        ).trim()

                        echo "Previous Helm revision: ${previousRevision}"

                        try {
                            sh """
                                # Create namespace if it doesn't exist
                                kubectl create namespace ${PRODUCTION_NAMESPACE} \
                                    --dry-run=client -o yaml | kubectl apply -f -

                                # Deploy with production values
                                helm upgrade --install ${HELM_RELEASE_NAME} ${HELM_CHART_PATH} \
                                    --namespace ${PRODUCTION_NAMESPACE} \
                                    -f ${HELM_CHART_PATH}/values-production.yaml \
                                    --set image.repository=${DOCKER_IMAGE} \
                                    --set image.tag=${IMAGE_TAG} \
                                    --wait \
                                    --timeout 10m

                                # Verify rollout
                                kubectl rollout status deployment/${HELM_RELEASE_NAME} \
                                    -n ${PRODUCTION_NAMESPACE} \
                                    --timeout=180s
                            """

                            echo "Production deployment successful!"

                        } catch (Exception e) {
                            // ---------------------------------------------------------
                            // AUTOMATIC ROLLBACK ON FAILURE
                            // ---------------------------------------------------------
                            // If the deployment or health check fails, roll back to
                            // the previous Helm revision automatically.
                            // ---------------------------------------------------------
                            echo "Production deployment FAILED! Rolling back..."

                            if (previousRevision != '0') {
                                sh """
                                    helm rollback ${HELM_RELEASE_NAME} ${previousRevision} \
                                        -n ${PRODUCTION_NAMESPACE} \
                                        --wait \
                                        --timeout 5m
                                """
                                echo "Rolled back to revision ${previousRevision}"
                            } else {
                                echo "No previous revision to roll back to."
                            }

                            // Re-throw the exception to mark the build as failed
                            throw e
                        }
                    }
                }
            }
        }

        // =====================================================================
        // STAGE 10: PRODUCTION VERIFICATION
        // =====================================================================
        stage('Production Verification') {
            steps {
                script {
                    withCredentials([
                        file(credentialsId: KUBECONFIG_PRODUCTION_ID, variable: 'KUBECONFIG')
                    ]) {
                        sh """
                            echo '=== Pod Status ==='
                            kubectl get pods -n ${PRODUCTION_NAMESPACE} \
                                -l app.kubernetes.io/name=${APP_NAME}

                            echo '=== Deployment Details ==='
                            kubectl describe deployment ${HELM_RELEASE_NAME} \
                                -n ${PRODUCTION_NAMESPACE}

                            echo '=== Helm Release Info ==='
                            helm list -n ${PRODUCTION_NAMESPACE}
                        """
                    }
                }
            }
        }
    }

    // -------------------------------------------------------------------------
    // POST: Cleanup and notifications
    // -------------------------------------------------------------------------
    post {
        always {
            sh """
                docker rmi ${DOCKER_IMAGE}:${IMAGE_TAG} 2>/dev/null || true
                docker rmi ${DOCKER_IMAGE}:latest 2>/dev/null || true
            """
            cleanWs()
        }
        success {
            echo "Helm deployment of ${APP_NAME}:${IMAGE_TAG} completed successfully!"
        }
        failure {
            echo "Helm deployment of ${APP_NAME}:${IMAGE_TAG} FAILED!"
        }
    }
}
```

---

## 3. VM Deployment - Deep Dive

This section provides a step-by-step guide to deploying Docker containers on virtual machines using Jenkins.

### Setting Up the Target VM

#### Prerequisites

Before you can deploy to a VM, the following must be in place:

| Requirement | Why |
|-------------|-----|
| Linux VM (Ubuntu 20.04+ recommended) | Target deployment server |
| Docker installed on the VM | To run containers |
| SSH server running on the VM | Jenkins connects via SSH |
| A dedicated deploy user | Security best practice (never deploy as root) |
| Firewall configured | Allow only necessary ports |

#### Installing Docker on the VM

```bash
# Connect to your VM
ssh root@your-vm-ip

# Update package index
sudo apt update

# Install prerequisites
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Verify Docker is running
sudo systemctl status docker
docker --version
```

#### Creating a Deploy User

Never deploy as root. Create a dedicated user with limited permissions:

```bash
# Create the deploy user
sudo adduser deploy --disabled-password --gecos ""

# Add the deploy user to the docker group so it can run Docker commands
# without sudo. This is necessary for Jenkins to execute docker commands
# over SSH.
sudo usermod -aG docker deploy

# Verify the user is in the docker group
id deploy
# Output should include: groups=...,999(docker)

# Switch to the deploy user and verify Docker works
su - deploy
docker ps
# Should show an empty table (no permission errors)
```

#### Setting Up SSH Key-Based Authentication

Jenkins will authenticate to the VM using SSH keys (no passwords).

```bash
# ON YOUR LOCAL MACHINE OR JENKINS SERVER:
# Generate an SSH key pair specifically for deployments
ssh-keygen -t ed25519 -C "jenkins-deploy-key" -f ~/.ssh/jenkins_deploy_key -N ""

# This creates two files:
#   ~/.ssh/jenkins_deploy_key       (private key - goes into Jenkins)
#   ~/.ssh/jenkins_deploy_key.pub   (public key - goes on the VM)

# Copy the public key to the VM's deploy user
ssh-copy-id -i ~/.ssh/jenkins_deploy_key.pub deploy@your-vm-ip

# Test the connection (should not prompt for a password)
ssh -i ~/.ssh/jenkins_deploy_key deploy@your-vm-ip "echo 'SSH connection successful'"
```

If `ssh-copy-id` is not available, manually add the key:

```bash
# ON THE VM:
sudo mkdir -p /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh

# Paste the contents of jenkins_deploy_key.pub into authorized_keys
sudo nano /home/deploy/.ssh/authorized_keys
# (paste the public key, save, and exit)

sudo chmod 600 /home/deploy/.ssh/authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh
```

#### Configuring Firewall Rules

```bash
# ON THE VM:
# Allow SSH (port 22)
sudo ufw allow 22/tcp

# Allow HTTP traffic (port 80) - this is where the app will be exposed
sudo ufw allow 80/tcp

# Allow HTTPS traffic (port 443) if using SSL
sudo ufw allow 443/tcp

# Enable the firewall
sudo ufw enable

# Check the rules
sudo ufw status verbose
```

### Jenkins Configuration for VM Deploy

#### Installing the SSH Agent Plugin

1. Go to **Jenkins Dashboard** > **Manage Jenkins** > **Manage Plugins**.
2. Click the **Available** tab.
3. Search for **SSH Agent Plugin**.
4. Check the box and click **Install without restart**.
5. Restart Jenkins if prompted.

#### Adding SSH Credentials in Jenkins

1. Go to **Jenkins Dashboard** > **Manage Jenkins** > **Manage Credentials**.
2. Click on the appropriate credentials store (usually **Jenkins** > **Global credentials**).
3. Click **Add Credentials**.
4. Fill in:
   - **Kind**: SSH Username with private key
   - **Scope**: Global
   - **ID**: `vm-ssh-key` (this ID is referenced in the Jenkinsfile)
   - **Description**: SSH key for VM deployment
   - **Username**: `deploy`
   - **Private Key**: Select "Enter directly" and paste the contents of `~/.ssh/jenkins_deploy_key`
   - **Passphrase**: Leave empty (or enter if you set one during key generation)
5. Click **OK**.

#### Testing SSH Connectivity from Jenkins

Create a quick test pipeline to verify Jenkins can SSH into the VM:

```groovy
pipeline {
    agent any
    stages {
        stage('Test SSH') {
            steps {
                sshagent(credentials: ['vm-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no deploy@192.168.1.100 \
                            'echo "SSH connection from Jenkins successful!" && hostname && docker --version'
                    """
                }
            }
        }
    }
}
```

### Deployment Script

For more complex deployments, use a dedicated deployment script on the VM rather than inline SSH commands. This script handles pulling, stopping, starting, health checking, and rolling back.

#### deploy.sh

```bash
#!/bin/bash
# =============================================================================
# deploy.sh - Docker Deployment Script for VM
# =============================================================================
# Usage: ./deploy.sh <image> <container_name> <host_port> <container_port> [env]
#
# Example:
#   ./deploy.sh mycompany/flask-app:abc1234 flask-app 80 5000 production
#
# This script:
#   1. Pulls the specified Docker image
#   2. Records the current running image (for rollback)
#   3. Stops and removes the old container
#   4. Starts a new container
#   5. Runs a health check
#   6. Rolls back automatically if the health check fails
# =============================================================================

set -euo pipefail

# ---------------------------------------------------------------------------
# ARGUMENTS
# ---------------------------------------------------------------------------
IMAGE="${1:?Usage: $0 <image> <container_name> <host_port> <container_port> [env]}"
CONTAINER_NAME="${2:?Missing container name}"
HOST_PORT="${3:?Missing host port}"
CONTAINER_PORT="${4:?Missing container port}"
ENVIRONMENT="${5:-production}"

# ---------------------------------------------------------------------------
# CONFIGURATION
# ---------------------------------------------------------------------------
HEALTH_CHECK_URL="http://localhost:${HOST_PORT}/health"
HEALTH_CHECK_RETRIES=12          # Number of health check attempts
HEALTH_CHECK_INTERVAL=5          # Seconds between attempts
ROLLBACK_FILE="/tmp/${CONTAINER_NAME}_previous_image.txt"
LOG_FILE="/var/log/deployments/${CONTAINER_NAME}.log"

# Ensure log directory exists
mkdir -p "$(dirname "${LOG_FILE}")"

# ---------------------------------------------------------------------------
# LOGGING FUNCTION
# ---------------------------------------------------------------------------
log() {
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[${timestamp}] $1" | tee -a "${LOG_FILE}"
}

# ---------------------------------------------------------------------------
# STEP 1: PULL THE NEW IMAGE
# ---------------------------------------------------------------------------
log "=== Starting deployment of ${IMAGE} ==="
log "Environment: ${ENVIRONMENT}"
log "Container: ${CONTAINER_NAME}"
log "Ports: ${HOST_PORT}:${CONTAINER_PORT}"

log "Pulling image: ${IMAGE}"
if ! docker pull "${IMAGE}"; then
    log "ERROR: Failed to pull image ${IMAGE}"
    exit 1
fi

# ---------------------------------------------------------------------------
# STEP 2: SAVE CURRENT IMAGE FOR ROLLBACK
# ---------------------------------------------------------------------------
PREVIOUS_IMAGE="none"
if docker inspect "${CONTAINER_NAME}" > /dev/null 2>&1; then
    PREVIOUS_IMAGE=$(docker inspect --format='{{.Config.Image}}' "${CONTAINER_NAME}")
    echo "${PREVIOUS_IMAGE}" > "${ROLLBACK_FILE}"
    log "Previous image recorded: ${PREVIOUS_IMAGE}"
else
    log "No existing container found. Fresh deployment."
fi

# ---------------------------------------------------------------------------
# STEP 3: STOP AND REMOVE OLD CONTAINER
# ---------------------------------------------------------------------------
log "Stopping old container..."
docker stop "${CONTAINER_NAME}" 2>/dev/null || true
docker rm "${CONTAINER_NAME}" 2>/dev/null || true

# ---------------------------------------------------------------------------
# STEP 4: START NEW CONTAINER
# ---------------------------------------------------------------------------
log "Starting new container..."
docker run -d \
    --name "${CONTAINER_NAME}" \
    --restart unless-stopped \
    -p "${HOST_PORT}:${CONTAINER_PORT}" \
    -e "FLASK_ENV=${ENVIRONMENT}" \
    -e "APP_VERSION=$(echo "${IMAGE}" | cut -d: -f2)" \
    --memory="512m" \
    --cpus="1.0" \
    --log-driver=json-file \
    --log-opt max-size=50m \
    --log-opt max-file=3 \
    "${IMAGE}"

log "Container started. Waiting for it to initialize..."

# ---------------------------------------------------------------------------
# STEP 5: HEALTH CHECK
# ---------------------------------------------------------------------------
log "Running health checks (max ${HEALTH_CHECK_RETRIES} attempts)..."
HEALTHY=false

for i in $(seq 1 "${HEALTH_CHECK_RETRIES}"); do
    log "Health check attempt ${i}/${HEALTH_CHECK_RETRIES}..."

    if curl -sf --max-time 5 "${HEALTH_CHECK_URL}" > /dev/null 2>&1; then
        log "Health check PASSED on attempt ${i}"
        HEALTHY=true
        break
    fi

    log "Health check not yet passing. Waiting ${HEALTH_CHECK_INTERVAL}s..."
    sleep "${HEALTH_CHECK_INTERVAL}"
done

# ---------------------------------------------------------------------------
# STEP 6: ROLLBACK IF UNHEALTHY
# ---------------------------------------------------------------------------
if [ "${HEALTHY}" = false ]; then
    log "ERROR: Health check failed after ${HEALTH_CHECK_RETRIES} attempts!"
    log "Container logs:"
    docker logs --tail 50 "${CONTAINER_NAME}" 2>&1 | tee -a "${LOG_FILE}"

    if [ "${PREVIOUS_IMAGE}" != "none" ]; then
        log "Rolling back to ${PREVIOUS_IMAGE}..."
        docker stop "${CONTAINER_NAME}" 2>/dev/null || true
        docker rm "${CONTAINER_NAME}" 2>/dev/null || true

        docker run -d \
            --name "${CONTAINER_NAME}" \
            --restart unless-stopped \
            -p "${HOST_PORT}:${CONTAINER_PORT}" \
            -e "FLASK_ENV=${ENVIRONMENT}" \
            "${PREVIOUS_IMAGE}"

        log "Rollback complete. Running: ${PREVIOUS_IMAGE}"
    else
        log "No previous image available for rollback."
    fi

    exit 1
fi

# ---------------------------------------------------------------------------
# STEP 7: CLEANUP OLD IMAGES
# ---------------------------------------------------------------------------
log "Cleaning up unused Docker images..."
docker image prune -f > /dev/null 2>&1 || true

# ---------------------------------------------------------------------------
# DONE
# ---------------------------------------------------------------------------
log "=== Deployment successful! ==="
log "Image: ${IMAGE}"
log "Container: ${CONTAINER_NAME}"
log "URL: http://localhost:${HOST_PORT}"
```

Make the script executable and place it on the VM:

```bash
# Copy the script to the VM
scp deploy.sh deploy@your-vm-ip:~/deploy.sh

# On the VM, make it executable
ssh deploy@your-vm-ip 'chmod +x ~/deploy.sh'
```

To use this script from Jenkins:

```groovy
stage('Deploy to Production') {
    steps {
        sshagent(credentials: ['vm-ssh-key']) {
            sh """
                ssh -o StrictHostKeyChecking=no deploy@${PRODUCTION_VM} \
                    './deploy.sh ${DOCKER_IMAGE}:${IMAGE_TAG} ${APP_NAME} ${HOST_PORT} ${APP_PORT} production'
            """
        }
    }
}
```

### Docker Compose on VM

For applications with multiple services (e.g., web app + database + cache), Docker Compose is the preferred approach for VM deployments.

#### docker-compose.yml

```yaml
# docker-compose.yml
# This file defines a multi-service deployment for the Flask application.
# Services: web app, PostgreSQL database, Redis cache, Nginx reverse proxy.

version: '3.8'

services:

  # ---------------------------------------------------------------------------
  # FLASK WEB APPLICATION
  # ---------------------------------------------------------------------------
  app:
    image: mycompany/flask-app:latest    # Updated by CI/CD pipeline
    container_name: flask-app
    restart: unless-stopped
    environment:
      - FLASK_ENV=production
      - DATABASE_URL=postgresql://appuser:secret@db:5432/appdb
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # ---------------------------------------------------------------------------
  # POSTGRESQL DATABASE
  # ---------------------------------------------------------------------------
  db:
    image: postgres:15-alpine
    container_name: flask-db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=appdb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ---------------------------------------------------------------------------
  # REDIS CACHE
  # ---------------------------------------------------------------------------
  redis:
    image: redis:7-alpine
    container_name: flask-redis
    restart: unless-stopped
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ---------------------------------------------------------------------------
  # NGINX REVERSE PROXY
  # ---------------------------------------------------------------------------
  nginx:
    image: nginx:1.25-alpine
    container_name: flask-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - app
    networks:
      - app-network

# ---------------------------------------------------------------------------
# VOLUMES: Persistent storage
# ---------------------------------------------------------------------------
volumes:
  postgres_data:
    driver: local

# ---------------------------------------------------------------------------
# NETWORKS: Isolated network for the services
# ---------------------------------------------------------------------------
networks:
  app-network:
    driver: bridge
```

#### Deploying with Docker Compose from Jenkins

```groovy
stage('Deploy with Docker Compose') {
    steps {
        sshagent(credentials: ['vm-ssh-key']) {
            sh """
                # Copy the updated docker-compose.yml to the VM
                scp -o StrictHostKeyChecking=no docker-compose.yml \
                    deploy@${PRODUCTION_VM}:~/app/docker-compose.yml

                # SSH in, pull new images, and restart
                ssh -o StrictHostKeyChecking=no deploy@${PRODUCTION_VM} << 'ENDSSH'
                    cd ~/app

                    # Update the image tag in docker-compose.yml
                    # Using sed to replace the image tag
                    sed -i "s|mycompany/flask-app:.*|mycompany/flask-app:${IMAGE_TAG}|" \
                        docker-compose.yml

                    # Pull the updated images
                    docker compose pull

                    # Restart only the services with updated images
                    # --no-deps prevents restarting dependent services
                    docker compose up -d --no-deps app

                    # Wait for the app to be healthy
                    echo "Waiting for health check..."
                    sleep 15

                    # Verify all services are running
                    docker compose ps

                    # Check health
                    curl -f http://localhost/health || exit 1

                    echo "Docker Compose deployment successful!"
ENDSSH
            """
        }
    }
}
```

---

## 4. Kubernetes Deployment - Deep Dive

### Kubernetes Basics (Quick Overview)

**Kubernetes** (often abbreviated as **K8s**) is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications. Originally developed by Google, it is now maintained by the Cloud Native Computing Foundation (CNCF).

#### Key Concepts

| Concept | Description |
|---------|-------------|
| **Pod** | The smallest deployable unit in Kubernetes. A pod contains one or more containers that share storage and network. |
| **Deployment** | Manages a set of identical pods. Handles rolling updates, scaling, and self-healing (restarting failed pods). |
| **Service** | A stable network endpoint that routes traffic to pods. Pods come and go, but the Service address stays the same. |
| **Namespace** | A virtual cluster within a physical cluster. Used to isolate environments (dev, staging, prod) or teams. |
| **ConfigMap** | Stores non-sensitive configuration data as key-value pairs. Injected into pods as environment variables or files. |
| **Secret** | Like ConfigMap, but for sensitive data (passwords, API keys). Values are base64-encoded (not encrypted by default). |
| **Ingress** | Manages external HTTP/HTTPS access to services. Acts as a reverse proxy with routing rules. |
| **Node** | A physical or virtual machine in the cluster that runs pods. |
| **ReplicaSet** | Ensures a specified number of pod replicas are running. Deployments manage ReplicaSets automatically. |

#### How They Relate

```
                     Ingress (external traffic)
                         |
                     Service (stable endpoint)
                         |
                    Deployment (manages replicas)
                    /    |    \
                Pod    Pod    Pod    (actual containers)
```

#### kubectl Basics

`kubectl` is the command-line tool for interacting with Kubernetes clusters.

```bash
# View cluster info
kubectl cluster-info

# List all pods in the current namespace
kubectl get pods

# List all pods in all namespaces
kubectl get pods --all-namespaces

# List all services
kubectl get services

# List all deployments
kubectl get deployments

# Get detailed information about a resource
kubectl describe pod <pod-name>

# View logs for a pod
kubectl logs <pod-name>

# Execute a command inside a pod
kubectl exec -it <pod-name> -- /bin/sh

# Apply a configuration file
kubectl apply -f manifest.yaml

# Delete a resource
kubectl delete -f manifest.yaml
```

### Installing kubectl

#### macOS

```bash
# Using Homebrew (recommended)
brew install kubectl

# Or download directly
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client
```

#### Linux

```bash
# Using snap
sudo snap install kubectl --classic

# Or download directly
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Using apt (Debian/Ubuntu)
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubectl

# Verify
kubectl version --client
```

#### Windows

```powershell
# Using Chocolatey
choco install kubernetes-cli

# Using Scoop
scoop install kubectl

# Or download directly
curl.exe -LO "https://dl.k8s.io/release/v1.29.0/bin/windows/amd64/kubectl.exe"
# Move kubectl.exe to a directory in your PATH

# Verify
kubectl version --client
```

### Setting Up a Local Kubernetes Cluster

#### Minikube Installation

Minikube runs a single-node Kubernetes cluster on your local machine, perfect for learning and development.

**macOS:**

```bash
# Install with Homebrew
brew install minikube

# Or download directly
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube
```

**Linux:**

```bash
# Download and install
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

**Windows:**

```powershell
# Using Chocolatey
choco install minikube

# Using Scoop
scoop install minikube

# Or download the installer from:
# https://storage.googleapis.com/minikube/releases/latest/minikube-installer.exe
```

#### Starting a Cluster

```bash
# Start a Minikube cluster with default settings
minikube start

# Start with specific resources (recommended for real workloads)
minikube start --cpus=2 --memory=4096 --disk-size=20g

# Start with a specific Kubernetes version
minikube start --kubernetes-version=v1.29.0

# Start with a specific driver (docker, virtualbox, hyperkit, etc.)
minikube start --driver=docker
```

#### Verifying the Cluster

```bash
# Check Minikube status
minikube status

# Check kubectl can connect to the cluster
kubectl cluster-info

# List nodes (should show one node)
kubectl get nodes

# Expected output:
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.29.0

# View all system pods (should all be Running)
kubectl get pods -n kube-system
```

### Kubernetes Manifests

Below are complete manifest files for deploying the Python Flask application to Kubernetes. Each field is explained.

#### namespace.yaml

```yaml
# namespace.yaml
# A Namespace provides a scope for resources. Using namespaces lets you
# run multiple environments (dev, staging, prod) in the same cluster
# without them interfering with each other.
apiVersion: v1              # API version for Namespace resource
kind: Namespace             # The type of Kubernetes resource
metadata:
  name: flask-app           # Name of the namespace. All our resources will
                            # be created in this namespace.
  labels:                   # Labels are key-value pairs for organizing resources
    app: flask-app          # Label identifying this namespace belongs to our app
    environment: dev        # Label for the environment
```

Apply it:

```bash
kubectl apply -f namespace.yaml
```

#### configmap.yaml

```yaml
# configmap.yaml
# ConfigMaps store non-sensitive configuration data. The application reads
# these values as environment variables or mounted files. This keeps
# configuration separate from the container image.
apiVersion: v1              # API version for ConfigMap
kind: ConfigMap             # Resource type
metadata:
  name: flask-app-config   # Name referenced by the Deployment
  namespace: flask-app     # Must match the namespace we created
  labels:
    app: flask-app
data:
  # Each key-value pair becomes an environment variable in the pod.
  FLASK_ENV: "development"               # Flask environment mode
  FLASK_APP: "app.py"                    # Entry point for the Flask app
  LOG_LEVEL: "INFO"                      # Application log level
  DATABASE_HOST: "postgres-service"      # Hostname of the database service
  DATABASE_PORT: "5432"                  # Database port
  DATABASE_NAME: "appdb"                 # Database name
```

#### secret.yaml

```yaml
# secret.yaml
# Secrets store sensitive data like passwords, API keys, and tokens.
# Values MUST be base64-encoded. In production, use a secrets manager
# (Vault, AWS Secrets Manager, etc.) instead of storing secrets in YAML files.
#
# WARNING: This file contains sensitive data. Do NOT commit it to version
# control. Add it to .gitignore. This is shown here for learning purposes only.
apiVersion: v1              # API version for Secret
kind: Secret                # Resource type
metadata:
  name: flask-app-secret   # Name referenced by the Deployment
  namespace: flask-app
  labels:
    app: flask-app
type: Opaque               # "Opaque" is the default type for arbitrary secrets.
                           # Other types: kubernetes.io/tls, kubernetes.io/dockerconfigjson
data:
  # Values are base64-encoded. To encode a value:
  #   echo -n 'mypassword' | base64
  # To decode:
  #   echo 'bXlwYXNzd29yZA==' | base64 -d
  DATABASE_USER: YXBwdXNlcg==          # base64 of "appuser"
  DATABASE_PASSWORD: c2VjcmV0cGFzcw==  # base64 of "secretpass"
  SECRET_KEY: bXlzdXBlcnNlY3JldGtleQ== # base64 of "mysupersecretkey"
```

To create a secret from the command line (avoids base64 encoding manually):

```bash
kubectl create secret generic flask-app-secret \
    --namespace flask-app \
    --from-literal=DATABASE_USER=appuser \
    --from-literal=DATABASE_PASSWORD=secretpass \
    --from-literal=SECRET_KEY=mysupersecretkey
```

#### deployment.yaml

```yaml
# deployment.yaml
# A Deployment manages a set of identical pods, ensuring the desired number
# of replicas are running at all times. It handles rolling updates, scaling,
# and self-healing (restarting crashed pods).
apiVersion: apps/v1         # API version for Deployment (apps group, version v1)
kind: Deployment            # Resource type
metadata:
  name: flask-app          # Name of the Deployment
  namespace: flask-app     # Namespace to deploy into
  labels:
    app: flask-app         # Labels for this Deployment resource
spec:
  # ---------------------------------------------------------------------------
  # REPLICA CONFIGURATION
  # ---------------------------------------------------------------------------
  replicas: 3              # Number of identical pod copies to maintain.
                           # Kubernetes will ensure exactly 3 pods are running.

  # ---------------------------------------------------------------------------
  # SELECTOR
  # ---------------------------------------------------------------------------
  # The selector tells the Deployment which pods it manages.
  # It must match the labels in the pod template below.
  selector:
    matchLabels:
      app: flask-app       # Manage pods with this label

  # ---------------------------------------------------------------------------
  # UPDATE STRATEGY
  # ---------------------------------------------------------------------------
  strategy:
    type: RollingUpdate    # Update pods gradually (not all at once)
    rollingUpdate:
      maxSurge: 1          # Allow 1 extra pod during updates (total = replicas + 1)
      maxUnavailable: 0    # Never allow fewer than 'replicas' healthy pods.
                           # This ensures zero downtime during updates.

  # ---------------------------------------------------------------------------
  # POD TEMPLATE
  # ---------------------------------------------------------------------------
  # This template defines what each pod looks like.
  template:
    metadata:
      labels:
        app: flask-app     # Must match spec.selector.matchLabels above
    spec:
      # -----------------------------------------------------------------------
      # CONTAINERS
      # -----------------------------------------------------------------------
      containers:
        - name: flask-app              # Container name (for logs, exec, etc.)
          image: mycompany/flask-app:latest  # Docker image to run.
                                             # In CD, the tag is updated per build.

          # --------------------------------------------------------------------
          # PORTS
          # --------------------------------------------------------------------
          ports:
            - containerPort: 5000      # Port the app listens on inside the container
              protocol: TCP

          # --------------------------------------------------------------------
          # ENVIRONMENT VARIABLES FROM CONFIGMAP
          # --------------------------------------------------------------------
          envFrom:
            - configMapRef:
                name: flask-app-config  # Injects all keys from the ConfigMap
                                        # as environment variables

          # --------------------------------------------------------------------
          # ENVIRONMENT VARIABLES FROM SECRET
          # --------------------------------------------------------------------
          env:
            - name: DATABASE_USER
              valueFrom:
                secretKeyRef:
                  name: flask-app-secret  # Reference to the Secret
                  key: DATABASE_USER      # Key within the Secret
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: flask-app-secret
                  key: DATABASE_PASSWORD
            - name: SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: flask-app-secret
                  key: SECRET_KEY

          # --------------------------------------------------------------------
          # RESOURCE LIMITS AND REQUESTS
          # --------------------------------------------------------------------
          # Requests: The minimum resources guaranteed to the container.
          # Limits: The maximum resources the container can use.
          resources:
            requests:
              cpu: "100m"       # 100 millicores = 0.1 CPU core
              memory: "128Mi"   # 128 mebibytes of RAM
            limits:
              cpu: "500m"       # Max 0.5 CPU core
              memory: "256Mi"   # Max 256 MiB RAM. Container is killed if it exceeds this.

          # --------------------------------------------------------------------
          # HEALTH CHECKS
          # --------------------------------------------------------------------
          # Liveness probe: Is the container alive? If it fails, Kubernetes
          # restarts the container.
          livenessProbe:
            httpGet:
              path: /health             # HTTP endpoint to check
              port: 5000                # Port to check on
            initialDelaySeconds: 30     # Wait 30s after start before first check
            periodSeconds: 10           # Check every 10 seconds
            timeoutSeconds: 5           # Timeout for each check
            failureThreshold: 3         # Restart after 3 consecutive failures

          # Readiness probe: Is the container ready to receive traffic?
          # If it fails, the pod is removed from Service endpoints (no traffic).
          readinessProbe:
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 10     # Wait 10s before first readiness check
            periodSeconds: 5            # Check every 5 seconds
            timeoutSeconds: 3           # Timeout for each check
            failureThreshold: 3         # Mark unready after 3 consecutive failures

          # Startup probe: Used for slow-starting containers. Until this passes,
          # liveness and readiness probes are disabled.
          startupProbe:
            httpGet:
              path: /health
              port: 5000
            failureThreshold: 30        # Allow up to 30 * 10s = 300s for startup
            periodSeconds: 10

      # -----------------------------------------------------------------------
      # RESTART POLICY
      # -----------------------------------------------------------------------
      restartPolicy: Always            # Always restart crashed containers
```

#### service.yaml

```yaml
# service.yaml
# A Service provides a stable network endpoint for accessing pods. Since pods
# are ephemeral (they can be created, destroyed, moved), their IP addresses
# change. A Service gives you a fixed IP and DNS name that routes to healthy pods.
#
# This file defines two Services: ClusterIP (internal) and NodePort (external).
---
# ---------------------------------------------------------------------------
# CLUSTERIP SERVICE
# ---------------------------------------------------------------------------
# ClusterIP is the default Service type. It exposes the service on an internal
# IP within the cluster. Other pods can reach it, but external traffic cannot.
# Use this when another service inside the cluster (e.g., Nginx, API gateway)
# forwards traffic to your app.
apiVersion: v1
kind: Service
metadata:
  name: flask-app          # Service name. Other pods use this as a DNS hostname:
                           #   http://flask-app.flask-app.svc.cluster.local:80
  namespace: flask-app
  labels:
    app: flask-app
spec:
  type: ClusterIP          # Only accessible within the cluster
  selector:
    app: flask-app         # Route traffic to pods with this label.
                           # Must match the pod template labels in the Deployment.
  ports:
    - name: http
      protocol: TCP
      port: 80             # Port the Service listens on (other pods use this)
      targetPort: 5000     # Port on the pod to forward to (the container port)
---
# ---------------------------------------------------------------------------
# NODEPORT SERVICE (for external access without an Ingress)
# ---------------------------------------------------------------------------
# NodePort exposes the service on a static port on every node in the cluster.
# External users can access it via <node-ip>:<node-port>.
# Use this for development/testing or when you do not have an Ingress controller.
apiVersion: v1
kind: Service
metadata:
  name: flask-app-nodeport
  namespace: flask-app
  labels:
    app: flask-app
spec:
  type: NodePort           # Accessible from outside the cluster via node IP
  selector:
    app: flask-app         # Route to pods with this label
  ports:
    - name: http
      protocol: TCP
      port: 80             # Port within the cluster
      targetPort: 5000     # Port on the container
      nodePort: 30080      # External port on every node (range: 30000-32767).
                           # Access via: http://<node-ip>:30080
```

#### ingress.yaml

```yaml
# ingress.yaml
# An Ingress manages external HTTP(S) access to services in the cluster.
# It acts as a reverse proxy, routing requests based on hostnames and paths.
# Requires an Ingress Controller to be installed (e.g., Nginx Ingress Controller).
apiVersion: networking.k8s.io/v1   # API version for Ingress
kind: Ingress
metadata:
  name: flask-app-ingress
  namespace: flask-app
  labels:
    app: flask-app
  annotations:
    # These annotations configure the Nginx Ingress Controller.
    # Different controllers use different annotations.
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    # Rate limiting (optional)
    nginx.ingress.kubernetes.io/limit-rps: "10"
spec:
  ingressClassName: nginx        # Name of the IngressClass to use.
                                 # Must match an installed Ingress Controller.
  rules:
    # -----------------------------------------------------------------------
    # ROUTING RULE: Route traffic for flask-app.example.com to our service
    # -----------------------------------------------------------------------
    - host: flask-app.example.com   # The hostname this rule applies to.
                                     # DNS must point to the cluster/load balancer.
      http:
        paths:
          - path: /                  # Match all paths starting with /
            pathType: Prefix         # "Prefix" matches /foo, /foo/bar, etc.
                                     # "Exact" matches only the exact path.
            backend:
              service:
                name: flask-app      # Route to this Service (ClusterIP service above)
                port:
                  number: 80         # The port on the Service to forward to
  # -----------------------------------------------------------------------
  # TLS CONFIGURATION (optional, for HTTPS)
  # -----------------------------------------------------------------------
  # tls:
  #   - hosts:
  #       - flask-app.example.com
  #     secretName: flask-app-tls    # Kubernetes Secret containing the TLS cert/key
```

### Deploying with kubectl

#### Applying Manifests

```bash
# Apply all manifests at once (Kubernetes resolves dependencies automatically)
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml

# Or apply an entire directory of manifests
kubectl apply -f ./k8s/

# Apply with a specific namespace override
kubectl apply -f deployment.yaml -n my-namespace
```

#### Checking Deployment Status

```bash
# Check the deployment status
kubectl get deployment flask-app -n flask-app

# Expected output:
# NAME        READY   UP-TO-DATE   AVAILABLE   AGE
# flask-app   3/3     3            3           2m

# Watch the rollout in real-time (useful during updates)
kubectl rollout status deployment/flask-app -n flask-app

# View rollout history
kubectl rollout history deployment/flask-app -n flask-app
```

#### Viewing Pods, Logs, and Resources

```bash
# List pods with extra details (node, IP, status)
kubectl get pods -n flask-app -o wide

# View logs for a specific pod
kubectl logs flask-app-7d4b8c6f5-abc12 -n flask-app

# Follow logs in real-time (like tail -f)
kubectl logs -f flask-app-7d4b8c6f5-abc12 -n flask-app

# View logs for all pods with a specific label
kubectl logs -l app=flask-app -n flask-app

# Describe a pod (shows events, conditions, and configuration)
kubectl describe pod flask-app-7d4b8c6f5-abc12 -n flask-app

# View all resources in the namespace
kubectl get all -n flask-app
```

#### Port-Forwarding for Testing

Port-forwarding lets you access a pod or service from your local machine without exposing it externally. Great for debugging.

```bash
# Forward local port 8080 to pod port 5000
kubectl port-forward pod/flask-app-7d4b8c6f5-abc12 8080:5000 -n flask-app

# Forward local port 8080 to Service port 80
kubectl port-forward service/flask-app 8080:80 -n flask-app

# Now access the app at http://localhost:8080

# Forward in the background
kubectl port-forward service/flask-app 8080:80 -n flask-app &
```

#### Scaling Deployments

```bash
# Scale to 5 replicas
kubectl scale deployment flask-app --replicas=5 -n flask-app

# Verify scaling
kubectl get pods -n flask-app
# You should see 5 pods

# Scale down to 2 replicas
kubectl scale deployment flask-app --replicas=2 -n flask-app

# Autoscaling (requires metrics-server)
kubectl autoscale deployment flask-app \
    --min=2 --max=10 --cpu-percent=80 -n flask-app
```

#### Rolling Updates and Rollbacks

```bash
# Update the image (triggers a rolling update)
kubectl set image deployment/flask-app \
    flask-app=mycompany/flask-app:v2.0.0 -n flask-app

# Watch the rolling update progress
kubectl rollout status deployment/flask-app -n flask-app

# View rollout history
kubectl rollout history deployment/flask-app -n flask-app

# Roll back to the previous version
kubectl rollout undo deployment/flask-app -n flask-app

# Roll back to a specific revision
kubectl rollout undo deployment/flask-app --to-revision=2 -n flask-app

# Pause a rollout (useful to inspect mid-update)
kubectl rollout pause deployment/flask-app -n flask-app

# Resume a paused rollout
kubectl rollout resume deployment/flask-app -n flask-app
```

---

## 5. Helm Deployment - Deep Dive

### What is Helm

**Helm** is the package manager for Kubernetes. Just as `apt` manages packages on Ubuntu or `brew` on macOS, Helm manages Kubernetes applications.

#### Core Concepts

| Concept | Description |
|---------|-------------|
| **Chart** | A package of Kubernetes manifest templates and default values. A chart defines everything needed to deploy an application. |
| **Release** | A running instance of a chart in a cluster. You can install the same chart multiple times with different configurations (e.g., dev release, staging release). |
| **Repository** | A collection of charts available for download. Like Docker Hub for containers, but for Helm charts. |
| **Values** | Configuration parameters that customize a chart. Override defaults to change replicas, image tags, resource limits, etc. |

#### Why Use Helm Over Raw Manifests

| Raw Manifests | Helm Charts |
|---------------|-------------|
| Hardcoded values in every file | Templated values, configure once in `values.yaml` |
| Copy-paste to deploy to different environments | One chart, multiple value files (`values-dev.yaml`, `values-prod.yaml`) |
| Manual version tracking | Built-in release versioning and history |
| No rollback mechanism | `helm rollback` to any previous release |
| Must apply/delete each file individually | `helm install` / `helm uninstall` manages everything as a unit |
| No dependency management | Chart dependencies (e.g., include PostgreSQL chart) |

### Installing Helm

#### macOS

```bash
# Using Homebrew (recommended)
brew install helm

# Verify
helm version
```

#### Linux

```bash
# Using the official install script (recommended)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Using snap
sudo snap install helm --classic

# Using apt (Debian/Ubuntu)
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt update
sudo apt install helm

# Verify
helm version
```

#### Windows

```powershell
# Using Chocolatey
choco install kubernetes-helm

# Using Scoop
scoop install helm

# Verify
helm version
```

#### Verifying Installation

```bash
helm version
# Output should show something like:
# version.BuildInfo{Version:"v3.14.0", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.21.6"}

# Add the official stable chart repository (useful for dependencies)
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### Creating a Helm Chart

#### Step 1: Generate the Chart Scaffold

```bash
# Create a new chart called "flask-app"
helm create flask-app

# This creates the following directory structure:
# flask-app/
# ├── Chart.yaml            # Metadata about the chart (name, version, description)
# ├── values.yaml           # Default configuration values
# ├── charts/               # Directory for chart dependencies
# ├── templates/            # Kubernetes manifest templates
# │   ├── _helpers.tpl      # Template helper functions (reusable snippets)
# │   ├── deployment.yaml   # Deployment template
# │   ├── service.yaml      # Service template
# │   ├── ingress.yaml      # Ingress template
# │   ├── hpa.yaml          # HorizontalPodAutoscaler template
# │   ├── serviceaccount.yaml
# │   ├── NOTES.txt         # Post-install instructions shown to the user
# │   └── tests/
# │       └── test-connection.yaml
# └── .helmignore           # Files to exclude when packaging the chart
```

#### Step 2: Chart.yaml

```yaml
# Chart.yaml
# This file contains metadata about the Helm chart. It tells Helm the chart's
# name, version, and other identifying information.
apiVersion: v2               # Helm chart API version. v2 is for Helm 3.

name: flask-app              # The name of the chart.

description: >
  A Helm chart for deploying the Flask web application to Kubernetes.
  Includes deployment, service, ingress, configmap, and secret management.

# Chart version. Increment this every time you change the chart itself
# (templates, values structure, etc.). Follows Semantic Versioning.
version: 0.1.0

# Application version. This is the version of the app being deployed.
# Updated automatically by the CD pipeline (or manually).
appVersion: "1.0.0"

# Type: "application" is a deployable chart. "library" is a reusable helper.
type: application

# Keywords for searching in Helm repositories
keywords:
  - flask
  - python
  - web

# Maintainers of this chart
maintainers:
  - name: DevOps Team
    email: devops@example.com

# Chart dependencies (e.g., if your app needs PostgreSQL)
# dependencies:
#   - name: postgresql
#     version: "12.x.x"
#     repository: "https://charts.bitnami.com/bitnami"
#     condition: postgresql.enabled
```

#### Step 3: values.yaml

```yaml
# values.yaml
# Default configuration values for the flask-app chart.
# These values can be overridden during install/upgrade using:
#   --set key=value
#   -f custom-values.yaml
#
# Every value here is accessible in templates via {{ .Values.<key> }}

# ---------------------------------------------------------------------------
# IMAGE CONFIGURATION
# ---------------------------------------------------------------------------
image:
  repository: mycompany/flask-app    # Docker image name (without tag)
  tag: "latest"                       # Image tag. Overridden by CD pipeline.
  pullPolicy: IfNotPresent            # When to pull the image:
                                      #   Always: Always pull (good for :latest)
                                      #   IfNotPresent: Only pull if not cached
                                      #   Never: Never pull (use local image)

# Image pull secrets (for private registries)
imagePullSecrets: []
# imagePullSecrets:
#   - name: docker-registry-secret

# ---------------------------------------------------------------------------
# REPLICA CONFIGURATION
# ---------------------------------------------------------------------------
replicaCount: 3                       # Number of pod replicas

# ---------------------------------------------------------------------------
# SERVICE CONFIGURATION
# ---------------------------------------------------------------------------
service:
  type: ClusterIP                     # Service type: ClusterIP, NodePort, LoadBalancer
  port: 80                            # Port the service exposes
  targetPort: 5000                    # Port on the container

# ---------------------------------------------------------------------------
# INGRESS CONFIGURATION
# ---------------------------------------------------------------------------
ingress:
  enabled: false                      # Set to true to create an Ingress resource
  className: "nginx"                  # IngressClass name
  annotations: {}
    # nginx.ingress.kubernetes.io/rewrite-target: /
  hosts:
    - host: flask-app.local           # Hostname for the ingress rule
      paths:
        - path: /
          pathType: Prefix
  tls: []
  #  - secretName: flask-app-tls
  #    hosts:
  #      - flask-app.local

# ---------------------------------------------------------------------------
# RESOURCE LIMITS
# ---------------------------------------------------------------------------
resources:
  requests:
    cpu: "100m"                       # Minimum CPU guaranteed
    memory: "128Mi"                   # Minimum memory guaranteed
  limits:
    cpu: "500m"                       # Maximum CPU allowed
    memory: "256Mi"                   # Maximum memory allowed

# ---------------------------------------------------------------------------
# AUTOSCALING
# ---------------------------------------------------------------------------
autoscaling:
  enabled: false                      # Set to true to enable HPA
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

# ---------------------------------------------------------------------------
# HEALTH CHECKS
# ---------------------------------------------------------------------------
healthCheck:
  path: /health                       # Health check endpoint
  port: 5000

livenessProbe:
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3

# ---------------------------------------------------------------------------
# APPLICATION CONFIGURATION (injected as environment variables)
# ---------------------------------------------------------------------------
config:
  FLASK_ENV: "production"
  FLASK_APP: "app.py"
  LOG_LEVEL: "INFO"

# ---------------------------------------------------------------------------
# SECRETS (injected as environment variables from a Kubernetes Secret)
# ---------------------------------------------------------------------------
secrets:
  DATABASE_USER: "appuser"
  DATABASE_PASSWORD: "changeme"
  SECRET_KEY: "changeme"

# ---------------------------------------------------------------------------
# NODE SELECTOR, TOLERATIONS, AFFINITY
# ---------------------------------------------------------------------------
nodeSelector: {}

tolerations: []

affinity: {}

# ---------------------------------------------------------------------------
# SERVICE ACCOUNT
# ---------------------------------------------------------------------------
serviceAccount:
  create: true
  annotations: {}
  name: ""
```

#### Step 4: templates/_helpers.tpl

```yaml
{{/*
templates/_helpers.tpl
Helper template functions used across all templates.
These avoid repeating the same logic in every template file.
*/}}

{{/*
Expand the name of the chart.
Used as a base for resource naming.
*/}}
{{- define "flask-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a fully qualified app name.
We truncate at 63 chars because Kubernetes name fields are limited to this.
If release name contains chart name, it will be used as a full name.
*/}}
{{- define "flask-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "flask-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels applied to all resources.
These are the standard Kubernetes recommended labels.
*/}}
{{- define "flask-app.labels" -}}
helm.sh/chart: {{ include "flask-app.chart" . }}
{{ include "flask-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels. Used in spec.selector.matchLabels and spec.template.metadata.labels.
These MUST remain consistent across upgrades (Kubernetes enforces this).
*/}}
{{- define "flask-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "flask-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use.
*/}}
{{- define "flask-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "flask-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

#### Step 5: templates/configmap.yaml

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "flask-app.fullname" . }}-config
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
data:
  {{- range $key, $value := .Values.config }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
```

#### Step 6: templates/deployment.yaml

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "flask-app.fullname" . }}
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "flask-app.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      # Annotations with a checksum of the ConfigMap and Secret force a
      # pod restart when configuration changes (otherwise Kubernetes
      # would not know the config changed).
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "flask-app.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "flask-app.serviceAccountName" . }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          # Inject all ConfigMap values as environment variables
          envFrom:
            - configMapRef:
                name: {{ include "flask-app.fullname" . }}-config
          # Inject secrets as individual environment variables
          env:
            {{- range $key, $value := .Values.secrets }}
            - name: {{ $key }}
              valueFrom:
                secretKeyRef:
                  name: {{ include "flask-app.fullname" $ }}-secret
                  key: {{ $key }}
            {{- end }}
          # Liveness probe
          livenessProbe:
            httpGet:
              path: {{ .Values.healthCheck.path }}
              port: {{ .Values.healthCheck.port }}
            initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.livenessProbe.periodSeconds }}
            timeoutSeconds: {{ .Values.livenessProbe.timeoutSeconds }}
            failureThreshold: {{ .Values.livenessProbe.failureThreshold }}
          # Readiness probe
          readinessProbe:
            httpGet:
              path: {{ .Values.healthCheck.path }}
              port: {{ .Values.healthCheck.port }}
            initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.readinessProbe.periodSeconds }}
            timeoutSeconds: {{ .Values.readinessProbe.timeoutSeconds }}
            failureThreshold: {{ .Values.readinessProbe.failureThreshold }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

#### Step 7: templates/service.yaml

```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "flask-app.fullname" . }}
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
      name: http
  selector:
    {{- include "flask-app.selectorLabels" . | nindent 4 }}
```

#### Step 8: templates/ingress.yaml

```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "flask-app.fullname" . }}
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "flask-app.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

#### Step 9: templates/NOTES.txt

```
# templates/NOTES.txt
# This file is displayed to the user after `helm install` or `helm upgrade`.

=======================================================
  Flask App has been deployed!
=======================================================

Release:     {{ .Release.Name }}
Namespace:   {{ .Release.Namespace }}
App Version: {{ .Chart.AppVersion }}
Replicas:    {{ .Values.replicaCount }}
Image:       {{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}

{{- if .Values.ingress.enabled }}

The application is available at:
{{- range .Values.ingress.hosts }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ .host }}
{{- end }}

{{- else if contains "NodePort" .Values.service.type }}

Get the application URL by running:
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "flask-app.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT

{{- else if contains "ClusterIP" .Values.service.type }}

The app is running on ClusterIP. To access it locally, run:
  kubectl port-forward --namespace {{ .Release.Namespace }} svc/{{ include "flask-app.fullname" . }} 8080:{{ .Values.service.port }}
  echo "Visit http://127.0.0.1:8080"

{{- end }}

To check the status of the deployment:
  kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "flask-app.name" . }}"
```

### Using Helm

#### helm install - Deploying the Chart

```bash
# Install the chart with default values
# Syntax: helm install <release-name> <chart-path>
helm install flask-app ./helm/flask-app \
    --namespace flask-app \
    --create-namespace

# Install with custom values file
helm install flask-app ./helm/flask-app \
    --namespace flask-app \
    --create-namespace \
    -f ./helm/flask-app/values-staging.yaml

# Install with inline value overrides
helm install flask-app ./helm/flask-app \
    --namespace flask-app \
    --create-namespace \
    --set image.tag=abc1234 \
    --set replicaCount=5

# Install and wait for all pods to be ready
helm install flask-app ./helm/flask-app \
    --namespace flask-app \
    --create-namespace \
    --wait \
    --timeout 5m

# Dry run (see what would be created without actually creating it)
helm install flask-app ./helm/flask-app \
    --namespace flask-app \
    --dry-run
```

#### helm upgrade - Updating Deployments

```bash
# Upgrade the release with a new image tag
helm upgrade flask-app ./helm/flask-app \
    --namespace flask-app \
    --set image.tag=new-tag-123

# Upgrade with a values file
helm upgrade flask-app ./helm/flask-app \
    --namespace flask-app \
    -f ./helm/flask-app/values-production.yaml \
    --set image.tag=new-tag-123

# Upgrade or install if it does not exist (idempotent)
helm upgrade --install flask-app ./helm/flask-app \
    --namespace flask-app \
    --create-namespace

# Upgrade and wait for pods to be ready
helm upgrade flask-app ./helm/flask-app \
    --namespace flask-app \
    --wait \
    --timeout 5m
```

#### helm rollback - Rolling Back

```bash
# Roll back to the previous release
helm rollback flask-app -n flask-app

# Roll back to a specific revision number
helm rollback flask-app 3 -n flask-app

# Roll back and wait for completion
helm rollback flask-app 3 -n flask-app --wait --timeout 5m
```

#### helm list - Viewing Releases

```bash
# List all releases in a namespace
helm list -n flask-app

# List all releases across all namespaces
helm list --all-namespaces

# Show detailed output
helm list -n flask-app -o yaml
```

#### helm uninstall - Removing Releases

```bash
# Uninstall a release (removes all Kubernetes resources it created)
helm uninstall flask-app -n flask-app

# Uninstall but keep the release history (allows rollback even after uninstall)
helm uninstall flask-app -n flask-app --keep-history
```

#### helm template - Rendering Templates Locally

```bash
# Render templates without deploying (useful for debugging)
helm template flask-app ./helm/flask-app \
    --namespace flask-app \
    --set image.tag=abc1234

# Render and save to a file
helm template flask-app ./helm/flask-app \
    --namespace flask-app > rendered-manifests.yaml
```

#### helm lint - Validating Charts

```bash
# Lint the chart for errors and warnings
helm lint ./helm/flask-app

# Lint with specific values
helm lint ./helm/flask-app -f ./helm/flask-app/values-production.yaml
```

#### Values Overrides

Helm values can be overridden in several ways (in order of precedence, lowest to highest):

```bash
# 1. Default values.yaml in the chart (lowest priority)
#    (automatically used)

# 2. Override with a values file (-f flag)
helm install flask-app ./helm/flask-app \
    -f values-production.yaml

# 3. Override with --set (highest priority)
helm install flask-app ./helm/flask-app \
    -f values-production.yaml \
    --set image.tag=abc1234

# 4. Multiple value files (later files take precedence)
helm install flask-app ./helm/flask-app \
    -f values-production.yaml \
    -f values-secrets.yaml

# Common --set examples:
helm upgrade flask-app ./helm/flask-app \
    --set image.tag=abc1234 \
    --set replicaCount=5 \
    --set ingress.enabled=true \
    --set resources.limits.memory=512Mi
```

### Helm in Jenkins Pipeline

#### How the CD Jenkinsfile Uses Helm

The Kubernetes/Helm Jenkinsfile in Section 2 uses Helm for deployment. The key line is:

```groovy
helm upgrade --install flask-app ./helm/flask-app \
    --namespace production \
    -f ./helm/flask-app/values-production.yaml \
    --set image.repository=mycompany/flask-app \
    --set image.tag=${IMAGE_TAG} \
    --wait \
    --timeout 10m
```

This command:
1. `upgrade --install`: Upgrades the release if it exists, or installs it if it does not. This makes the pipeline idempotent.
2. `-f values-production.yaml`: Uses production-specific configuration.
3. `--set image.tag=${IMAGE_TAG}`: Overrides the image tag with the one just built by the pipeline.
4. `--wait`: Blocks until all pods are healthy.
5. `--timeout 10m`: Fails if pods are not ready within 10 minutes.

#### Installing Helm on Jenkins Agent

If your Jenkins agent does not have Helm pre-installed, install it in the pipeline:

```groovy
stage('Setup Tools') {
    steps {
        sh '''
            # Install Helm if not present
            if ! command -v helm &> /dev/null; then
                echo "Installing Helm..."
                curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
            fi
            helm version

            # Install kubectl if not present
            if ! command -v kubectl &> /dev/null; then
                echo "Installing kubectl..."
                curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                chmod +x kubectl
                sudo mv kubectl /usr/local/bin/
            fi
            kubectl version --client
        '''
    }
}
```

#### kubeconfig Management in Jenkins

The kubeconfig file tells kubectl and Helm how to connect to the Kubernetes cluster. Store it securely in Jenkins credentials.

**Adding kubeconfig to Jenkins:**

1. Go to **Manage Jenkins** > **Manage Credentials**.
2. Click **Add Credentials**.
3. Select **Kind**: Secret file.
4. Upload your kubeconfig file.
5. Set **ID** to `kubeconfig-production` (or `kubeconfig-staging`).

**Using it in the pipeline:**

```groovy
withCredentials([file(credentialsId: 'kubeconfig-production', variable: 'KUBECONFIG')]) {
    sh '''
        # KUBECONFIG env var is automatically set to the path of the temp file.
        # kubectl and helm will use it automatically.
        kubectl get nodes
        helm list --all-namespaces
    '''
}
```

---

## 6. Environment Management

### Environment-Specific Values Files

Create separate values files for each environment. Keep them in the Helm chart directory.

#### values-dev.yaml

```yaml
# values-dev.yaml
# Development environment configuration.
# Fewer replicas, more verbose logging, no ingress.
replicaCount: 1

image:
  tag: "latest"
  pullPolicy: Always       # Always pull in dev to get latest changes

config:
  FLASK_ENV: "development"
  LOG_LEVEL: "DEBUG"

resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "200m"
    memory: "128Mi"

ingress:
  enabled: false

autoscaling:
  enabled: false
```

#### values-staging.yaml

```yaml
# values-staging.yaml
# Staging environment. Mirrors production but with fewer resources.
replicaCount: 2

image:
  pullPolicy: IfNotPresent

config:
  FLASK_ENV: "staging"
  LOG_LEVEL: "INFO"

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"

ingress:
  enabled: true
  hosts:
    - host: flask-app.staging.example.com
      paths:
        - path: /
          pathType: Prefix

autoscaling:
  enabled: false
```

#### values-production.yaml

```yaml
# values-production.yaml
# Production environment. Maximum reliability and resources.
replicaCount: 3

image:
  pullPolicy: IfNotPresent

config:
  FLASK_ENV: "production"
  LOG_LEVEL: "WARNING"

resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1000m"
    memory: "512Mi"

ingress:
  enabled: true
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: flask-app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: flask-app-tls
      hosts:
        - flask-app.example.com

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

### Namespace-Based Environments

Using namespaces to isolate environments within the same cluster:

```bash
# Create namespaces for each environment
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace production

# Deploy to dev
helm upgrade --install flask-app ./helm/flask-app \
    --namespace dev \
    -f ./helm/flask-app/values-dev.yaml

# Deploy to staging
helm upgrade --install flask-app ./helm/flask-app \
    --namespace staging \
    -f ./helm/flask-app/values-staging.yaml

# Deploy to production
helm upgrade --install flask-app ./helm/flask-app \
    --namespace production \
    -f ./helm/flask-app/values-production.yaml
```

Advantages of namespace isolation:
- Each environment has its own set of resources (pods, services, configmaps).
- Resource quotas can be set per namespace to prevent one environment from consuming all cluster resources.
- RBAC (Role-Based Access Control) can restrict who has access to each namespace.

```yaml
# resource-quota.yaml
# Limit resources available to the dev namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "20"
```

### Secrets Management Across Environments

Never store production secrets in plain text in version control. Here are approaches for managing secrets:

**Approach 1: Jenkins Credentials + --set**

Store secrets in Jenkins and inject them during deployment:

```groovy
withCredentials([
    string(credentialsId: 'prod-db-password', variable: 'DB_PASS'),
    string(credentialsId: 'prod-secret-key', variable: 'SECRET_KEY')
]) {
    sh """
        helm upgrade --install flask-app ./helm/flask-app \
            --namespace production \
            -f values-production.yaml \
            --set secrets.DATABASE_PASSWORD=\${DB_PASS} \
            --set secrets.SECRET_KEY=\${SECRET_KEY}
    """
}
```

**Approach 2: Kubernetes Secrets Created Separately**

Create secrets outside of Helm and reference them in the deployment:

```bash
# Create the secret directly in Kubernetes
kubectl create secret generic flask-app-secret \
    --namespace production \
    --from-literal=DATABASE_PASSWORD=supersecretpassword \
    --from-literal=SECRET_KEY=anothersecretkey
```

**Approach 3: External Secrets Operator**

Use tools like External Secrets Operator to sync secrets from AWS Secrets Manager, HashiCorp Vault, or Azure Key Vault into Kubernetes Secrets automatically. This is the recommended approach for production.

---

## 7. Monitoring Deployments

### kubectl Commands for Monitoring

```bash
# ----- Pod Status -----
# Quick overview of all pods
kubectl get pods -n production

# Watch pods in real-time (updates every 2 seconds)
kubectl get pods -n production -w

# Detailed pod information including events
kubectl describe pod <pod-name> -n production

# ----- Deployment Status -----
# Check if the deployment is healthy
kubectl get deployment flask-app -n production

# Watch a rollout in progress
kubectl rollout status deployment/flask-app -n production

# View rollout history
kubectl rollout history deployment/flask-app -n production

# ----- Resource Usage -----
# View CPU and memory usage per pod (requires metrics-server)
kubectl top pods -n production

# View node-level resource usage
kubectl top nodes
```

### Checking Rollout Status

```bash
# Check current rollout status (blocks until complete or failed)
kubectl rollout status deployment/flask-app -n production

# Possible outputs:
#   "deployment "flask-app" successfully rolled out"       --> SUCCESS
#   "Waiting for deployment ... rollout to finish: ..."    --> IN PROGRESS
#   (timeout or error)                                      --> FAILED

# View detailed rollout history with change cause
kubectl rollout history deployment/flask-app -n production
# REVISION  CHANGE-CAUSE
# 1         Initial deployment
# 2         Updated image to v1.1.0
# 3         Updated image to v1.2.0

# View details of a specific revision
kubectl rollout history deployment/flask-app --revision=2 -n production
```

### Viewing Events and Logs

```bash
# ----- Events -----
# View recent events in the namespace (useful for debugging)
kubectl get events -n production --sort-by='.lastTimestamp'

# View events for a specific pod
kubectl describe pod <pod-name> -n production | grep -A 20 "Events:"

# ----- Logs -----
# View current logs
kubectl logs <pod-name> -n production

# Follow logs in real-time
kubectl logs -f <pod-name> -n production

# View logs from a previous (crashed) container
kubectl logs <pod-name> -n production --previous

# View logs from all pods with a label
kubectl logs -l app.kubernetes.io/name=flask-app -n production

# View last 100 lines
kubectl logs --tail=100 <pod-name> -n production

# View logs from the last hour
kubectl logs --since=1h <pod-name> -n production
```

### Basic Health Checks

```bash
# ----- From Outside the Cluster -----
# If using NodePort or LoadBalancer service:
curl -sf http://<external-ip>/health && echo "HEALTHY" || echo "UNHEALTHY"

# ----- From Inside the Cluster -----
# Run a temporary pod to test the service internally:
kubectl run health-check --rm -it --restart=Never \
    --image=curlimages/curl:latest \
    -n production \
    -- curl -sf http://flask-app:80/health

# ----- Using Port-Forward -----
# Forward and check locally:
kubectl port-forward svc/flask-app 8080:80 -n production &
curl -sf http://localhost:8080/health && echo "HEALTHY" || echo "UNHEALTHY"

# ----- Helm Release Status -----
helm status flask-app -n production

# ----- Comprehensive Check Script -----
# A quick check you can run after any deployment:
echo "=== Pods ===" && \
kubectl get pods -n production -l app.kubernetes.io/name=flask-app && \
echo "" && \
echo "=== Deployment ===" && \
kubectl get deployment flask-app -n production && \
echo "" && \
echo "=== Service ===" && \
kubectl get svc flask-app -n production && \
echo "" && \
echo "=== Recent Events ===" && \
kubectl get events -n production --sort-by='.lastTimestamp' | tail -10
```

---

## 8. Practice Exercises

### Exercise 1: Deploy to a VM with Docker

**Objective**: Set up a complete VM deployment pipeline manually (no Jenkins, just the concepts).

**Steps**:

1. Set up a VM (use a local VM with Vagrant, or a cloud instance):
   ```bash
   # If using Vagrant (install Vagrant and VirtualBox first):
   mkdir vm-deploy-exercise && cd vm-deploy-exercise
   vagrant init ubuntu/jammy64
   vagrant up
   vagrant ssh
   ```

2. Install Docker on the VM:
   ```bash
   sudo apt update
   sudo apt install -y docker.io
   sudo usermod -aG docker $USER
   # Log out and back in for group change to take effect
   exit
   vagrant ssh
   ```

3. Create a simple Flask app locally:
   ```bash
   mkdir flask-app && cd flask-app

   cat > app.py << 'EOF'
   from flask import Flask, jsonify
   app = Flask(__name__)

   @app.route('/')
   def home():
       return jsonify({"message": "Hello from VM deployment!"})

   @app.route('/health')
   def health():
       return jsonify({"status": "healthy"}), 200

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   EOF

   cat > requirements.txt << 'EOF'
   flask==3.0.0
   EOF

   cat > Dockerfile << 'EOF'
   FROM python:3.11-slim
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY . .
   EXPOSE 5000
   CMD ["python", "app.py"]
   EOF
   ```

4. Build and test locally:
   ```bash
   docker build -t flask-app:v1 .
   docker run -d --name test -p 5000:5000 flask-app:v1
   curl http://localhost:5000/health
   docker stop test && docker rm test
   ```

5. Transfer the image to the VM and run it:
   ```bash
   # Save the image to a tar file
   docker save flask-app:v1 > flask-app-v1.tar

   # Copy to VM (if using Vagrant)
   vagrant scp flask-app-v1.tar :~/flask-app-v1.tar

   # On the VM, load and run
   vagrant ssh
   docker load < ~/flask-app-v1.tar
   docker run -d --name flask-app -p 80:5000 flask-app:v1
   curl http://localhost/health
   ```

6. Simulate an update (deploy v2):
   ```bash
   # Modify app.py to return a different message
   # Rebuild as flask-app:v2
   # Transfer to VM
   # Stop v1, start v2
   ```

### Exercise 2: Deploy to Minikube with kubectl

**Objective**: Deploy the Flask app to a local Kubernetes cluster using raw manifests.

**Steps**:

1. Start Minikube:
   ```bash
   minikube start
   kubectl cluster-info
   ```

2. Build the image inside Minikube (so it is available to the cluster):
   ```bash
   # Point your Docker client to Minikube's Docker daemon
   eval $(minikube docker-env)

   # Build the image (it will be available inside Minikube)
   docker build -t flask-app:v1 .
   ```

3. Create the namespace:
   ```bash
   kubectl create namespace exercise
   ```

4. Create the Deployment manifest (`deployment.yaml`):
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: flask-app
     namespace: exercise
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: flask-app
     template:
       metadata:
         labels:
           app: flask-app
       spec:
         containers:
           - name: flask-app
             image: flask-app:v1
             imagePullPolicy: Never    # Use the local image, don't pull
             ports:
               - containerPort: 5000
   ```

5. Create the Service manifest (`service.yaml`):
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: flask-app
     namespace: exercise
   spec:
     type: NodePort
     selector:
       app: flask-app
     ports:
       - port: 80
         targetPort: 5000
         nodePort: 30080
   ```

6. Deploy and verify:
   ```bash
   kubectl apply -f deployment.yaml
   kubectl apply -f service.yaml
   kubectl get pods -n exercise
   kubectl get svc -n exercise

   # Access the app
   minikube service flask-app -n exercise --url
   # Or
   curl http://$(minikube ip):30080/health
   ```

7. Scale and update:
   ```bash
   # Scale to 5 replicas
   kubectl scale deployment flask-app --replicas=5 -n exercise
   kubectl get pods -n exercise

   # Build v2 and do a rolling update
   # (modify app.py first, rebuild as flask-app:v2)
   eval $(minikube docker-env)
   docker build -t flask-app:v2 .
   kubectl set image deployment/flask-app flask-app=flask-app:v2 -n exercise
   kubectl rollout status deployment/flask-app -n exercise
   ```

8. Rollback:
   ```bash
   kubectl rollout undo deployment/flask-app -n exercise
   kubectl rollout status deployment/flask-app -n exercise
   ```

### Exercise 3: Create and Deploy a Helm Chart

**Objective**: Create a Helm chart from scratch and deploy it.

**Steps**:

1. Create the chart:
   ```bash
   helm create my-flask-chart
   cd my-flask-chart
   ```

2. Edit `values.yaml` to match the Flask app:
   ```bash
   # Change image.repository to flask-app
   # Change image.tag to v1
   # Change service.port to 80
   # Add service.targetPort: 5000
   ```

3. Lint the chart:
   ```bash
   helm lint .
   ```

4. Render the templates locally to verify:
   ```bash
   helm template my-release . --namespace exercise
   ```

5. Install the chart:
   ```bash
   helm install my-release . \
       --namespace exercise \
       --create-namespace \
       --set image.pullPolicy=Never
   ```

6. Verify:
   ```bash
   helm list -n exercise
   kubectl get all -n exercise
   ```

7. Upgrade with a new image tag:
   ```bash
   helm upgrade my-release . \
       --namespace exercise \
       --set image.tag=v2 \
       --set image.pullPolicy=Never
   ```

8. Rollback:
   ```bash
   helm rollback my-release 1 -n exercise
   ```

9. Clean up:
   ```bash
   helm uninstall my-release -n exercise
   ```

### Exercise 4: Multi-Environment Deployment

**Objective**: Deploy the same Helm chart to dev and staging namespaces with different configurations.

**Steps**:

1. Create two values files:
   ```bash
   # values-dev.yaml
   cat > values-dev.yaml << 'EOF'
   replicaCount: 1
   image:
     tag: v1
     pullPolicy: Never
   config:
     FLASK_ENV: development
     LOG_LEVEL: DEBUG
   resources:
     requests:
       cpu: 50m
       memory: 64Mi
     limits:
       cpu: 200m
       memory: 128Mi
   EOF

   # values-staging.yaml
   cat > values-staging.yaml << 'EOF'
   replicaCount: 2
   image:
     tag: v1
     pullPolicy: Never
   config:
     FLASK_ENV: staging
     LOG_LEVEL: INFO
   resources:
     requests:
       cpu: 100m
       memory: 128Mi
     limits:
       cpu: 500m
       memory: 256Mi
   EOF
   ```

2. Deploy to dev:
   ```bash
   helm upgrade --install flask-dev ./my-flask-chart \
       --namespace dev \
       --create-namespace \
       -f values-dev.yaml
   ```

3. Deploy to staging:
   ```bash
   helm upgrade --install flask-staging ./my-flask-chart \
       --namespace staging \
       --create-namespace \
       -f values-staging.yaml
   ```

4. Compare the environments:
   ```bash
   echo "=== DEV ===" && kubectl get all -n dev
   echo "=== STAGING ===" && kubectl get all -n staging
   ```

5. Update only staging:
   ```bash
   helm upgrade flask-staging ./my-flask-chart \
       --namespace staging \
       -f values-staging.yaml \
       --set image.tag=v2
   ```

6. Verify dev is still on v1 and staging is on v2:
   ```bash
   kubectl describe deployment -n dev | grep Image
   kubectl describe deployment -n staging | grep Image
   ```

### Exercise 5: Simulate a Failed Deployment and Rollback

**Objective**: Practice handling a deployment failure and rolling back.

**Steps**:

1. Deploy a working version:
   ```bash
   helm upgrade --install flask-app ./my-flask-chart \
       --namespace rollback-test \
       --create-namespace \
       --set image.tag=v1 \
       --set image.pullPolicy=Never
   ```

2. Verify it is healthy:
   ```bash
   kubectl get pods -n rollback-test
   helm list -n rollback-test
   ```

3. Deploy a "bad" version (use a non-existent image tag to simulate failure):
   ```bash
   helm upgrade flask-app ./my-flask-chart \
       --namespace rollback-test \
       --set image.tag=v999-does-not-exist \
       --set image.pullPolicy=Always \
       --wait \
       --timeout 60s || echo "Deployment failed as expected!"
   ```

4. Check the status (pods should be in ImagePullBackOff):
   ```bash
   kubectl get pods -n rollback-test
   kubectl describe pod -n rollback-test | grep -A5 "Events"
   helm history flask-app -n rollback-test
   ```

5. Roll back to the working version:
   ```bash
   helm rollback flask-app 1 -n rollback-test --wait
   ```

6. Verify recovery:
   ```bash
   kubectl get pods -n rollback-test
   helm list -n rollback-test
   ```

7. Clean up:
   ```bash
   helm uninstall flask-app -n rollback-test
   kubectl delete namespace rollback-test
   ```

---

## Next Steps

After completing this guide, continue learning with:

- **GitOps with ArgoCD or Flux**: Declarative continuous deployment using Git as the source of truth.
- **Advanced Kubernetes**: StatefulSets, DaemonSets, Jobs, CronJobs, Custom Resource Definitions.
- **Service Mesh**: Istio or Linkerd for advanced traffic management, observability, and security.
- **Monitoring**: Prometheus and Grafana for metrics, alerting, and dashboards.
- **Secrets Management**: HashiCorp Vault for production-grade secrets management.
