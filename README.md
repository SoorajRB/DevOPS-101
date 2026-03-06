# DevOps 101

## Introduction

Welcome to **DevOps 101**, a foundational guide designed to help you learn, practice, and master the essential concepts and tools used in modern DevOps workflows.

This repository is built for anyone starting their journey into DevOps — whether you're a developer, system administrator, student, or career switcher. The goal is to provide a hands-on, practical approach to learning DevOps by following a real-world pipeline from code to production.

## Learning Path

This repository follows a structured learning path that mirrors how software moves through a real DevOps pipeline:

```
Write Code → Containerize → Push to Registry → Deploy to VM
                                    ↓
                         Automate with CI/CD (Jenkins)
                                    ↓
                    Deploy to Kubernetes with Helm Charts
```

### Phase 1: Version Control with Git
Learn the fundamentals of tracking and collaborating on code.
- [Git Fundamentals](./01-version-control/git/README.md)
- [Git Practice Exercises](./01-version-control/git/01-practice.md)

### Phase 2: Containerization with Docker
Create a Python application, write a Dockerfile, build and run containers, and push images to a VM.
- [Docker Guide](./02-containerization/docker/README.md) — Covers the full chain:
  1. Creating a sample Python Flask application
  2. Writing a Dockerfile (single-stage and multi-stage)
  3. Building a Docker image locally
  4. Running the application in a container
  5. Pushing the Docker image to a VM (via registry and direct transfer)

### Phase 3: CI/CD with Jenkins
Automate the entire build-test-deploy cycle.
- [Jenkins CI Guide](./03-ci-cd/jenkins/README.md) — Covers:
  1. Installing and setting up Jenkins (macOS, Linux, Windows, Docker)
  2. Pushing code to GitHub
  3. Configuring Jenkins with GitHub (credentials, webhooks)
  4. Creating Jenkins Pipelines (Freestyle and Pipeline jobs)
  5. Writing and pushing a Jenkinsfile
  6. Configuring Jenkins-GitHub triggers (webhooks, Poll SCM, multibranch)
  7. Running and troubleshooting pipelines

- [CD Pipelines & Deployment Guide](./03-ci-cd/jenkins/CD-PIPELINES.md) — Covers:
  1. Writing CD pipelines in Jenkins
  2. VM deployment via SSH (with deployment scripts and rollback)
  3. Kubernetes deployment with Helm from Jenkins
  4. Environment management (dev, staging, production)
  5. Approval gates and deployment strategies

### Phase 4: Container Orchestration
Deploy and manage containers at scale.
- [Kubernetes Fundamentals](./04-orchestration/kubernetes/README.md) — Covers:
  1. Kubernetes architecture (control plane, worker nodes)
  2. Installing kubectl and Minikube
  3. Core concepts (Pods, Deployments, Services, Ingress, ConfigMaps, Secrets)
  4. Deploying the Python Flask app to Kubernetes
  5. Troubleshooting Kubernetes

- [Helm Package Manager](./04-orchestration/helm/README.md) — Covers:
  1. Installing Helm
  2. Creating Helm charts from scratch
  3. Template functions and syntax
  4. Managing multiple environments with values files
  5. Helm in CI/CD pipelines

## The Full Pipeline

Here's how all the pieces connect end-to-end:

```
Developer pushes code to GitHub
        ↓
GitHub webhook triggers Jenkins pipeline
        ↓
Jenkins runs the Jenkinsfile:
  1. Checks out code from GitHub
  2. Builds Docker image
  3. Runs tests
  4. Pushes image to Docker registry
  5. Deploys to target environment:
     ├── VM: SSH → docker pull → docker run
     └── Kubernetes: helm upgrade → rolling update
```

## Topics Covered

| Topic | Tool | Status |
|-------|------|--------|
| Version Control | Git | Available |
| Containerization | Docker | Available |
| CI Pipelines | Jenkins | Available |
| CD Pipelines | Jenkins | Available |
| Container Orchestration | Kubernetes | Available |
| Package Management | Helm | Available |
| GitHub Hosting | GitHub | Coming Soon |
| Infrastructure as Code | Terraform | Coming Soon |
| Configuration Management | Ansible | Coming Soon |
| Cloud Providers | AWS, GCP, Azure | Coming Soon |
| Monitoring & Logging | Prometheus, Grafana, ELK | Coming Soon |
| Security & Secrets | Vault, SOPS | Coming Soon |

## What You'll Find

Each section includes:
- Comprehensive documentation with detailed explanations
- Full installation steps for macOS, Linux, and Windows
- Code samples and complete configuration files
- Step-by-step guides you can follow from zero knowledge
- Practical exercises to reinforce learning

## Prerequisites

- Basic command line familiarity
- A computer running macOS, Linux, or Windows
- A GitHub account (free)
- Willingness to learn!

---

*Start your DevOps journey today — begin with [Git](./01-version-control/git/README.md), then work through each phase sequentially.*
