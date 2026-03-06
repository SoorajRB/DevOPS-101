# Helm - The Kubernetes Package Manager

## Table of Contents

- [1. Introduction to Helm](#1-introduction-to-helm)
- [2. Installing Helm](#2-installing-helm)
- [3. Helm Basics](#3-helm-basics)
- [4. Creating a Helm Chart from Scratch](#4-creating-a-helm-chart-from-scratch)
- [5. Helm Template Functions and Syntax](#5-helm-template-functions-and-syntax)
- [6. Testing and Validating Charts](#6-testing-and-validating-charts)
- [7. Advanced Helm Usage](#7-advanced-helm-usage)
- [8. Helm in CI/CD Pipelines](#8-helm-in-cicd-pipelines)
- [9. Helm Best Practices](#9-helm-best-practices)
- [10. Helm Commands Reference](#10-helm-commands-reference)
- [11. Practice Exercises](#11-practice-exercises)

---

## 1. Introduction to Helm

### What is Helm and Why Use It

Helm is the **package manager for Kubernetes**. Just as `apt` manages packages on Debian, `yum` on RHEL, or `brew` on macOS, Helm manages applications on Kubernetes clusters. It simplifies the process of defining, installing, upgrading, and managing even the most complex Kubernetes applications.

Without Helm, deploying an application to Kubernetes requires you to write and maintain multiple YAML manifest files -- Deployments, Services, ConfigMaps, Secrets, Ingresses, ServiceAccounts, RBAC rules, and more. As applications grow in complexity and as you need to deploy across multiple environments (development, staging, production), managing raw manifests becomes unwieldy, error-prone, and repetitive.

Helm solves these problems by packaging all related Kubernetes manifests into a single unit called a **chart**, with configurable parameters stored in a **values file**. This allows you to:

- **Templatize** your Kubernetes manifests so a single chart works across environments
- **Version** your application deployments and roll back when needed
- **Share** reusable application packages through chart repositories
- **Manage dependencies** between applications (e.g., an app that requires Redis and PostgreSQL)
- **Standardize** deployment practices across teams

### Helm vs Raw Kubernetes Manifests

| Aspect | Raw Manifests | Helm |
|---|---|---|
| **Reusability** | Copy-paste across environments | Single chart, multiple value files |
| **Parameterization** | Hardcoded values or manual find/replace | Template variables via `values.yaml` |
| **Versioning** | Manual tracking via Git | Built-in release versioning and history |
| **Rollback** | Manual `kubectl apply` of old files | `helm rollback` with one command |
| **Dependencies** | Managed manually | Declarative dependency management |
| **Sharing** | Share raw YAML files | Package and publish to chart repositories |
| **Lifecycle Hooks** | Not available | Pre/post install, upgrade, delete hooks |
| **Testing** | Manual verification | Built-in chart testing framework |
| **Complexity** | Grows linearly with resources | Managed via templates and helpers |

### Helm 3 vs Helm 2 (Brief History)

Helm has gone through significant evolution:

**Helm 2 (2016-2020):**
- Required a server-side component called **Tiller** running inside the Kubernetes cluster
- Tiller had broad permissions (cluster-admin), which was a major security concern
- Release information was stored in ConfigMaps managed by Tiller
- Used a two-tier architecture: Helm client communicating with Tiller server

**Helm 3 (2019-present):**
- **Removed Tiller entirely** -- Helm now communicates directly with the Kubernetes API server
- Uses the caller's kubeconfig credentials, integrating with Kubernetes RBAC naturally
- Release information is stored as **Secrets** in the release's namespace
- Improved upgrade strategy with three-way strategic merge patches
- Added support for OCI registries for chart storage
- JSON Schema validation for `values.yaml`
- Library charts (non-installable, reusable chart components)

**All examples in this guide use Helm 3**, which is the current standard.

### Key Concepts

#### Charts

A **chart** is a Helm package. It contains all of the resource definitions necessary to run an application, tool, or service inside a Kubernetes cluster. Think of it as the Kubernetes equivalent of a Homebrew formula, an APT dpkg, or an RPM file. A chart is a collection of files organized in a specific directory structure.

#### Releases

A **release** is a running instance of a chart combined with a specific configuration. One chart can be installed many times into the same cluster, and each time a new release is created. For example, you could install the MySQL chart twice -- once for your production database and once for your staging database -- each with its own release name.

#### Repositories

A **repository** (or repo) is a place where charts can be collected and shared. It is like a package registry (similar to npm for Node.js or PyPI for Python). Popular repositories include:

- **ArtifactHub** (https://artifacthub.io) -- the central hub for discovering charts
- **Bitnami** -- one of the most popular chart repositories
- **Ingress-NGINX** -- charts for the NGINX ingress controller

#### Values

**Values** provide a way to override the default configuration of a chart. Each chart has a `values.yaml` file that defines defaults. Users can provide their own values files or set individual values on the command line to customize the installation.

---

## 2. Installing Helm

### macOS

Using **Homebrew** (recommended):

```bash
brew install helm
```

Using **MacPorts**:

```bash
sudo port install helm-3
```

### Ubuntu/Debian

**Using APT (official Helm repository):**

```bash
# Add the Helm GPG key
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null

# Install apt-transport-https
sudo apt-get install apt-transport-https --yes

# Add the Helm repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

# Update and install
sudo apt-get update
sudo apt-get install helm
```

**Using Snap:**

```bash
sudo snap install helm --classic
```

### CentOS/RHEL

**Using DNF/YUM:**

```bash
# Add the Helm repository
cat <<EOF | sudo tee /etc/yum.repos.d/helm.repo
[helm-stable]
name=Helm Stable
baseurl=https://baltocdn.com/helm/stable/rpm/
enabled=1
gpgcheck=1
gpgkey=https://baltocdn.com/helm/signing.asc
EOF

# Install Helm
sudo dnf install helm
# or on older systems:
sudo yum install helm
```

### Windows

**Using Chocolatey:**

```powershell
choco install kubernetes-helm
```

**Using Scoop:**

```powershell
scoop install helm
```

**Using Winget:**

```powershell
winget install Helm.Helm
```

### From the Official Install Script (Any Linux/macOS)

This is a universal method that works on any Linux distribution or macOS:

```bash
# Download and run the install script
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Or as a single command:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### From Binary Releases (Manual Install)

```bash
# Download the latest release (adjust version and OS/arch as needed)
wget https://get.helm.sh/helm-v3.16.4-linux-amd64.tar.gz

# Extract the archive
tar -zxvf helm-v3.16.4-linux-amd64.tar.gz

# Move the binary to a location on your PATH
sudo mv linux-amd64/helm /usr/local/bin/helm

# Clean up
rm -rf linux-amd64 helm-v3.16.4-linux-amd64.tar.gz
```

### Verifying the Installation

After installing via any method, verify that Helm is working:

```bash
helm version
```

Expected output (version numbers will vary):

```
version.BuildInfo{Version:"v3.16.4", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.22.x"}
```

Check that Helm can reach your Kubernetes cluster:

```bash
# Ensure kubectl is configured first
kubectl cluster-info

# Then test Helm
helm list
```

If the cluster is reachable, `helm list` will return an empty table (no releases installed yet).

---

## 3. Helm Basics

### Helm Repositories

Repositories are where Helm charts are stored and shared. Before you can install charts from a repository, you must add it to your local Helm client.

#### Adding Repositories

```bash
# Add the Bitnami repository (one of the most popular)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Add the ingress-nginx repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Add the Prometheus community repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Add the Grafana repository
helm repo add grafana https://grafana.github.io/helm-charts

# Add the Jetstack repository (for cert-manager)
helm repo add jetstack https://charts.jetstack.io
```

#### Listing Repositories

```bash
helm repo list
```

Output:

```
NAME                    URL
bitnami                 https://charts.bitnami.com/bitnami
ingress-nginx           https://kubernetes.github.io/ingress-nginx
prometheus-community    https://prometheus-community.github.io/helm-charts
grafana                 https://grafana.github.io/helm-charts
jetstack                https://charts.jetstack.io
```

#### Updating Repositories

Update your local cache of chart information from all added repositories:

```bash
# Update all repositories
helm repo update

# Update a specific repository
helm repo update bitnami
```

You should run `helm repo update` periodically, especially before searching for or installing charts, to ensure you have the latest chart versions.

#### Searching for Charts

```bash
# Search for charts in your added repositories
helm search repo nginx

# Search with version information
helm search repo nginx --versions

# Search for a specific chart version
helm search repo bitnami/nginx --version 15.0.0

# Search on Artifact Hub (public charts from the internet)
helm search hub wordpress

# Search with output format
helm search repo redis --output yaml
```

Example output of `helm search repo nginx`:

```
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
bitnami/nginx                           15.4.4          1.25.4          NGINX is a high-performance HTTP server ...
bitnami/nginx-ingress-controller        11.0.0          1.10.0          NGINX Ingress Controller for Kubernetes ...
ingress-nginx/ingress-nginx             4.10.0          1.10.0          Ingress controller for Kubernetes using ...
```

#### Removing Repositories

```bash
helm repo remove bitnami
```

### Installing Charts

#### Basic Installation

The most basic form of installing a chart:

```bash
# helm install [RELEASE_NAME] [CHART]
helm install my-nginx bitnami/nginx
```

This creates a release named `my-nginx` from the `bitnami/nginx` chart with all default values.

#### Generating a Release Name

If you do not want to specify a name, use `--generate-name`:

```bash
helm install bitnami/nginx --generate-name
```

Helm will create a name like `nginx-1706123456`.

#### Installing with Custom Values

**Using a values file:**

```bash
helm install my-nginx bitnami/nginx -f my-values.yaml
```

**Setting individual values on the command line:**

```bash
helm install my-nginx bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=LoadBalancer \
  --set service.port=8080
```

**Combining a values file with overrides:**

```bash
helm install my-nginx bitnami/nginx \
  -f my-values.yaml \
  --set replicaCount=5
```

Values set with `--set` take precedence over those in `-f` files.

**Setting complex values:**

```bash
# Set nested values
--set ingress.enabled=true

# Set list items
--set ingress.hosts[0].host=example.com
--set ingress.hosts[0].paths[0].path=/
--set ingress.hosts[0].paths[0].pathType=Prefix

# Set string values (useful when values might be interpreted as numbers/booleans)
--set image.tag="1.0.0"
```

#### Installing in a Specific Namespace

```bash
# Install into an existing namespace
helm install my-nginx bitnami/nginx --namespace web-apps

# Create the namespace if it doesn't exist
helm install my-nginx bitnami/nginx --namespace web-apps --create-namespace
```

#### Installing a Specific Chart Version

```bash
helm install my-nginx bitnami/nginx --version 15.0.0
```

#### Dry Run and Debug

Before actually installing, preview what Helm will do:

```bash
# Dry run -- renders templates and shows output without installing
helm install my-nginx bitnami/nginx --dry-run

# Dry run with debug -- shows even more detail including computed values
helm install my-nginx bitnami/nginx --dry-run --debug
```

This is extremely useful for:
- Verifying your values produce the expected manifests
- Debugging template rendering issues
- Reviewing resources before they are created

#### Installing from a Local Chart Directory

```bash
helm install my-app ./my-chart/
```

#### Installing from a Packaged Chart

```bash
helm install my-app ./my-chart-1.0.0.tgz
```

#### Installing from a URL

```bash
helm install my-app https://example.com/charts/my-chart-1.0.0.tgz
```

#### Waiting for Resources

```bash
# Wait until all resources are in a ready state
helm install my-nginx bitnami/nginx --wait

# Wait with a custom timeout
helm install my-nginx bitnami/nginx --wait --timeout 5m
```

#### Viewing Default Values of a Chart

Before installing, inspect what values a chart accepts:

```bash
# Show all default values
helm show values bitnami/nginx

# Show chart metadata
helm show chart bitnami/nginx

# Show the README
helm show readme bitnami/nginx

# Show everything
helm show all bitnami/nginx
```

### Managing Releases

#### Listing Releases

```bash
# List releases in the current namespace
helm list

# List releases in all namespaces
helm list --all-namespaces
# or
helm list -A

# List releases in a specific namespace
helm list --namespace web-apps

# List all releases including failed, pending, etc.
helm list --all

# List with custom output format
helm list --output json
helm list --output yaml

# Filter releases by name
helm list --filter "nginx"
```

Example output:

```
NAME       NAMESPACE  REVISION  UPDATED                                STATUS    CHART          APP VERSION
my-nginx   default    1         2024-01-15 10:30:00.123456 +0000 UTC   deployed  nginx-15.4.4   1.25.4
my-redis   default    3         2024-01-14 08:15:00.654321 +0000 UTC   deployed  redis-18.6.1   7.2.4
```

#### Checking Release Status

```bash
helm status my-nginx
```

Output includes:
- Release name, namespace, revision number
- Deployment timestamp
- Status (deployed, failed, pending, etc.)
- Any notes from the chart's NOTES.txt

#### Getting Release Values

```bash
# Show the values that were used for the current release
helm get values my-nginx

# Show all computed values (defaults merged with overrides)
helm get values my-nginx --all

# Show the rendered manifests
helm get manifest my-nginx

# Show the release notes
helm get notes my-nginx

# Show all information about the release
helm get all my-nginx

# Show hooks
helm get hooks my-nginx
```

#### Upgrading Releases

```bash
# Upgrade with new values
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# Upgrade with a values file
helm upgrade my-nginx bitnami/nginx -f production-values.yaml

# Upgrade to a specific chart version
helm upgrade my-nginx bitnami/nginx --version 16.0.0

# Upgrade and install if the release doesn't exist
helm upgrade --install my-nginx bitnami/nginx

# Reuse the values from the previous release and merge new ones
helm upgrade my-nginx bitnami/nginx --reuse-values --set image.tag=1.26.0

# Reset values to defaults and apply only the provided values
helm upgrade my-nginx bitnami/nginx --reset-values -f new-values.yaml

# Upgrade with a dry run first
helm upgrade my-nginx bitnami/nginx --set replicaCount=5 --dry-run
```

**Important note about `--reuse-values`:** This flag carries forward all values from the previous release. If the chart has added new default values in a newer version, those defaults will not be applied. Use `--reset-values` combined with `-f` when upgrading chart versions.

#### Rolling Back Releases

```bash
# Roll back to the previous revision
helm rollback my-nginx

# Roll back to a specific revision number
helm rollback my-nginx 2

# Roll back with a dry run
helm rollback my-nginx 2 --dry-run

# Force resource updates during rollback
helm rollback my-nginx 2 --force
```

#### Viewing Release History

```bash
helm history my-nginx
```

Output:

```
REVISION  UPDATED                   STATUS       CHART          APP VERSION  DESCRIPTION
1         Mon Jan 15 10:30:00 2024  superseded   nginx-15.4.4   1.25.4       Install complete
2         Tue Jan 16 14:20:00 2024  superseded   nginx-15.4.4   1.25.4       Upgrade complete
3         Wed Jan 17 09:10:00 2024  deployed     nginx-16.0.0   1.26.0       Upgrade complete
```

#### Uninstalling Releases

```bash
# Uninstall a release
helm uninstall my-nginx

# Uninstall but keep the release history (allows rollback)
helm uninstall my-nginx --keep-history

# Uninstall from a specific namespace
helm uninstall my-nginx --namespace web-apps

# Dry run uninstall
helm uninstall my-nginx --dry-run
```

---

## 4. Creating a Helm Chart from Scratch

This section walks through creating a complete Helm chart for a Python Flask application. We will build every file from scratch with detailed explanations.

### Scaffolding a New Chart

Helm provides a command to generate a chart skeleton:

```bash
helm create flask-app
```

This creates the following directory structure:

```
flask-app/
  Chart.yaml            # Chart metadata (name, version, description)
  values.yaml           # Default configuration values
  charts/               # Directory for chart dependencies
  templates/            # Directory for Kubernetes manifest templates
    deployment.yaml     # Deployment template
    service.yaml        # Service template
    ingress.yaml        # Ingress template
    hpa.yaml            # HorizontalPodAutoscaler template
    serviceaccount.yaml # ServiceAccount template
    NOTES.txt           # Post-install/upgrade notes (displayed to user)
    _helpers.tpl        # Named template definitions (partials)
    tests/              # Test templates
      test-connection.yaml
  .helmignore           # Patterns to ignore when packaging
```

Let us now build each file with full explanations.

### Chart.yaml

The `Chart.yaml` file contains metadata about the chart. It is the only required file in a chart (aside from the `templates/` directory).

```yaml
# Chart.yaml - Metadata for the flask-app Helm chart

# apiVersion: v2 indicates this is a Helm 3 chart.
# Helm 2 charts used apiVersion: v1 and had slightly different fields.
apiVersion: v2

# name: The name of the chart. This should match the directory name.
# It is used as part of the release name and in template helpers.
name: flask-app

# description: A short, human-readable description of the chart.
description: A Helm chart for deploying a Python Flask web application on Kubernetes

# type: Can be "application" or "library".
#   - application: A chart that can be installed and creates Kubernetes resources.
#   - library: A chart that provides utilities/helpers to other charts but cannot be installed directly.
type: application

# version: The chart version. This follows Semantic Versioning (SemVer 2).
# This version represents the version of the chart itself, NOT the application.
# Increment this every time you change the chart (templates, values, etc.).
#   MAJOR.MINOR.PATCH
#   - MAJOR: Incompatible changes (e.g., renamed values, removed templates)
#   - MINOR: New features (e.g., added optional ingress support)
#   - PATCH: Bug fixes (e.g., fixed a template indentation issue)
version: 0.1.0

# appVersion: The version of the application being deployed by this chart.
# This is informational and does not affect chart behavior.
# It is displayed in `helm list` output and can be referenced in templates as .Chart.AppVersion.
appVersion: "1.0.0"

# keywords: Tags that help users find this chart when searching.
keywords:
  - flask
  - python
  - web
  - api

# home: The URL of the project's home page.
home: https://github.com/example/flask-app

# sources: A list of URLs to source code related to this chart.
sources:
  - https://github.com/example/flask-app

# maintainers: A list of people maintaining this chart.
maintainers:
  - name: DevOps Team
    email: devops@example.com
    url: https://example.com

# icon: A URL to an SVG or PNG image to use as an icon (optional).
# icon: https://example.com/icon.png

# dependencies: Other charts that this chart depends on (covered in Advanced section).
# dependencies:
#   - name: redis
#     version: "18.x.x"
#     repository: "https://charts.bitnami.com/bitnami"
#     condition: redis.enabled
```

**Key points:**
- `version` is the chart version -- bump this when you change anything in the chart
- `appVersion` is the application version -- bump this when the underlying app changes
- These two versions are independent of each other

### values.yaml

The `values.yaml` file defines the default configuration for the chart. Every configurable aspect of your deployment should be represented here. Users override these values when installing or upgrading.

```yaml
# values.yaml - Default configuration values for flask-app

# -- Number of pod replicas to deploy
replicaCount: 1

# -- Container image configuration
image:
  # -- Docker image repository (without tag)
  repository: myregistry.example.com/flask-app
  # -- Image pull policy. Use "Always" in production, "IfNotPresent" for development
  pullPolicy: IfNotPresent
  # -- Overrides the image tag. Defaults to the chart appVersion if not set.
  tag: ""

# -- List of Docker registry secrets for pulling private images
imagePullSecrets: []
#  - name: my-registry-secret

# -- Override the default name generated by the chart
nameOverride: ""

# -- Override the full release name generated by the chart
fullnameOverride: ""

# -- ServiceAccount configuration
serviceAccount:
  # -- Whether to create a ServiceAccount
  create: true
  # -- Annotations to add to the ServiceAccount
  annotations: {}
  # -- The name of the ServiceAccount. If not set, a name is generated using the fullname template.
  name: ""
  # -- Whether to automount the ServiceAccount token into pods
  automount: true

# -- Annotations to add to the Pod
podAnnotations: {}

# -- Labels to add to the Pod
podLabels: {}

# -- Pod security context (applied at the pod level)
podSecurityContext:
  fsGroup: 1000
  runAsNonRoot: true

# -- Container security context (applied at the container level)
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000

# -- Kubernetes Service configuration
service:
  # -- Service type: ClusterIP, NodePort, or LoadBalancer
  type: ClusterIP
  # -- Service port (the port the service exposes)
  port: 80
  # -- Container port (the port the application listens on inside the container)
  targetPort: 5000
  # -- NodePort number (only used when service.type is NodePort, range: 30000-32767)
  # nodePort: 30080

# -- Ingress configuration
ingress:
  # -- Whether to create an Ingress resource
  enabled: false
  # -- Ingress class name (e.g., "nginx", "traefik", "alb")
  className: "nginx"
  # -- Annotations to add to the Ingress
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # cert-manager.io/cluster-issuer: letsencrypt-prod
    # nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # nginx.ingress.kubernetes.io/proxy-body-size: "10m"
  # -- Ingress host configuration
  hosts:
    - host: flask-app.local
      paths:
        - path: /
          pathType: Prefix
  # -- TLS configuration for Ingress
  tls: []
  #  - secretName: flask-app-tls
  #    hosts:
  #      - flask-app.local

# -- Resource requests and limits for the container
resources:
  # -- Resource limits (maximum resources the container can use)
  limits:
    cpu: 500m
    memory: 256Mi
  # -- Resource requests (guaranteed resources for the container)
  requests:
    cpu: 100m
    memory: 128Mi

# -- Liveness probe configuration (restarts the container if it fails)
livenessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 10
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1

# -- Readiness probe configuration (removes the pod from service endpoints if it fails)
readinessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1

# -- Startup probe configuration (gives the container time to start before liveness checks begin)
startupProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 30

# -- Horizontal Pod Autoscaler configuration
autoscaling:
  # -- Whether to enable autoscaling
  enabled: false
  # -- Minimum number of replicas
  minReplicas: 2
  # -- Maximum number of replicas
  maxReplicas: 10
  # -- Target average CPU utilization percentage
  targetCPUUtilizationPercentage: 80
  # -- Target average memory utilization percentage
  targetMemoryUtilizationPercentage: 80

# -- Node selector for pod scheduling (key-value pairs)
nodeSelector: {}
  # disktype: ssd
  # kubernetes.io/os: linux

# -- Tolerations for pod scheduling
tolerations: []
  # - key: "dedicated"
  #   operator: "Equal"
  #   value: "web-apps"
  #   effect: "NoSchedule"

# -- Affinity rules for pod scheduling
affinity: {}
  # podAntiAffinity:
  #   preferredDuringSchedulingIgnoredDuringExecution:
  #     - weight: 100
  #       podAffinityTerm:
  #         labelSelector:
  #           matchExpressions:
  #             - key: app.kubernetes.io/name
  #               operator: In
  #               values:
  #                 - flask-app
  #         topologyKey: kubernetes.io/hostname

# -- Environment variables to set on the container
env:
  # -- Flask environment (development, testing, production)
  FLASK_ENV: "production"
  # -- Flask debug mode
  FLASK_DEBUG: "0"
  # -- Application log level
  LOG_LEVEL: "INFO"
  # -- Port the Flask application listens on
  PORT: "5000"

# -- Environment variables from Secrets (for sensitive data)
envFromSecrets: []
  # - name: DATABASE_URL
  #   secretName: flask-app-secrets
  #   secretKey: database-url
  # - name: API_KEY
  #   secretName: flask-app-secrets
  #   secretKey: api-key

# -- ConfigMap data to be mounted or used by the application
configMap:
  # -- Whether to create a ConfigMap
  enabled: true
  # -- ConfigMap key-value data
  data:
    APP_NAME: "Flask App"
    APP_VERSION: "1.0.0"
    WORKERS: "4"
    TIMEOUT: "120"

# -- Additional volume mounts
volumeMounts: []
  # - name: config-volume
  #   mountPath: /app/config
  #   readOnly: true

# -- Additional volumes
volumes: []
  # - name: config-volume
  #   configMap:
  #     name: flask-app-config
```

### Templates Deep Dive

Templates are the heart of a Helm chart. They are Kubernetes manifest files written with Go template syntax that get rendered into valid YAML when Helm processes them.

#### templates/_helpers.tpl

The `_helpers.tpl` file contains **named templates** (also called partials or sub-templates). These are reusable template snippets that can be included in other templates. The underscore prefix tells Helm not to render this file as a Kubernetes manifest.

```yaml
{{/*
Expand the name of the chart.
Uses the chart name from Chart.yaml, or the nameOverride value if set.
Truncated to 63 characters because Kubernetes name fields are limited to 63 chars.
*/}}
{{- define "flask-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this (by the DNS naming spec).
If the release name contains the chart name, it will be used as a full name.
This prevents names like "flask-app-flask-app" when the release is named "flask-app".
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
Example output: "flask-app-0.1.0"
*/}}
{{- define "flask-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels applied to all resources.
These labels follow the Kubernetes recommended labeling convention.
See: https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/
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
Selector labels used in both the Deployment spec.selector and the Service spec.selector.
These must match between the Deployment and Service for traffic routing to work.
*/}}
{{- define "flask-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "flask-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use.
If serviceAccount.create is true, use the generated name or the name from values.
If serviceAccount.create is false, use "default".
*/}}
{{- define "flask-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "flask-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

**Key Go template concepts used above:**

| Syntax | Meaning |
|---|---|
| `{{- ... -}}` | Trim whitespace before/after the output |
| `define "name"` | Define a named template |
| `include "name" .` | Include a named template, passing the current scope (`.`) |
| `.Chart.Name` | Access the `name` field from `Chart.yaml` |
| `.Values.nameOverride` | Access `nameOverride` from `values.yaml` |
| `.Release.Name` | The name given to the release during `helm install` |
| `.Release.Service` | Always "Helm" in Helm 3 |
| `default` | Returns the first non-empty value |
| `trunc 63` | Truncate string to 63 characters |
| `trimSuffix "-"` | Remove trailing hyphens |
| `contains` | Check if one string contains another |
| `printf` | Formatted string output |
| `replace` | String replacement |
| `quote` | Wraps value in double quotes |

#### templates/deployment.yaml

The Deployment template is typically the most complex template in a chart.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "flask-app.fullname" . }}
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  # Only set replicas if autoscaling is disabled.
  # When HPA is active, it manages the replica count.
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "flask-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        # Force a rolling update when the ConfigMap changes by including
        # a checksum of the ConfigMap data in the pod annotations.
        {{- if .Values.configMap.enabled }}
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- end }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "flask-app.labels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "flask-app.serviceAccountName" . }}
      {{- with .Values.podSecurityContext }}
      securityContext:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          {{- with .Values.securityContext }}
          securityContext:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          {{- with .Values.livenessProbe }}
          livenessProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.readinessProbe }}
          readinessProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.startupProbe }}
          startupProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
            {{- range .Values.envFromSecrets }}
            - name: {{ .name }}
              valueFrom:
                secretKeyRef:
                  name: {{ .secretName }}
                  key: {{ .secretKey }}
            {{- end }}
          {{- if .Values.configMap.enabled }}
          envFrom:
            - configMapRef:
                name: {{ include "flask-app.fullname" . }}-config
          {{- end }}
          {{- with .Values.volumeMounts }}
          volumeMounts:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with .Values.volumes }}
      volumes:
        {{- toYaml . | nindent 8 }}
      {{- end }}
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

**Template syntax breakdown:**

- **`{{ include "flask-app.fullname" . }}`** -- Calls the named template `flask-app.fullname` and inserts its output. The `.` passes the current scope (all chart data) to the template.

- **`{{- ... -}}`** -- The hyphens trim whitespace. `{{-` trims leading whitespace, `-}}` trims trailing whitespace. This prevents blank lines in the rendered YAML.

- **`{{ .Values.replicaCount }}`** -- Accesses the `replicaCount` field from `values.yaml`.

- **`{{- if not .Values.autoscaling.enabled }}`** -- Conditional: only render the `replicas` field if autoscaling is disabled.

- **`{{- with .Values.podAnnotations }}`** -- The `with` block changes the scope (`.`) to `.Values.podAnnotations`. If the value is empty/nil, the block is skipped entirely.

- **`{{- toYaml . | nindent 8 }}`** -- Converts the current scope to YAML and indents it by 8 spaces. `nindent` adds a newline before the indented content.

- **`{{- range $key, $value := .Values.env }}`** -- Iterates over the `env` map. `$key` and `$value` are loop variables.

- **`{{ $value | quote }}`** -- Pipes `$value` through the `quote` function, wrapping it in double quotes. This ensures values like `"true"` and `"123"` are treated as strings in YAML.

- **`{{ .Values.image.tag | default .Chart.AppVersion }}`** -- If `image.tag` is empty, falls back to `Chart.AppVersion`.

#### templates/service.yaml

```yaml
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
      targetPort: http
      protocol: TCP
      name: http
      {{- if and (eq .Values.service.type "NodePort") .Values.service.nodePort }}
      nodePort: {{ .Values.service.nodePort }}
      {{- end }}
  selector:
    {{- include "flask-app.selectorLabels" . | nindent 4 }}
```

**Notes:**
- The `targetPort: http` refers to the named port `http` defined in the Deployment's container ports section.
- The `nodePort` field is only rendered when the service type is `NodePort` and a specific port number is provided.
- The `selector` must match the labels on the pods created by the Deployment. Both use `flask-app.selectorLabels`.

#### templates/ingress.yaml

```yaml
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

**Notes:**
- The entire file is wrapped in `{{- if .Values.ingress.enabled -}}` so no Ingress resource is created when ingress is disabled.
- Inside a `range` loop, the scope (`.`) changes to the current item. To access the root scope, use `$` (e.g., `$.Values.service.port`).
- The `tls` and `hosts` sections use nested `range` loops to iterate over lists defined in `values.yaml`.

#### templates/configmap.yaml

```yaml
{{- if .Values.configMap.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "flask-app.fullname" . }}-config
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
data:
  {{- range $key, $value := .Values.configMap.data }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
{{- end }}
```

**Notes:**
- The ConfigMap is only created when `configMap.enabled` is `true`.
- The `range` loop iterates over the `configMap.data` map to produce key-value pairs.
- Values are quoted to ensure proper YAML string handling.

#### templates/hpa.yaml

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "flask-app.fullname" . }}
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "flask-app.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
```

**Notes:**
- Uses `autoscaling/v2` API (current stable version).
- The HPA is only created when `autoscaling.enabled` is `true`.
- CPU and memory metrics are each conditionally included only when their target percentage is set.
- When the HPA is active, the Deployment template omits the `replicas` field (as shown in the Deployment template above), because the HPA controls scaling.

#### templates/serviceaccount.yaml

```yaml
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "flask-app.serviceAccountName" . }}
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- if .Values.serviceAccount.automount }}
automountServiceAccountToken: true
{{- end }}
{{- end }}
```

**Notes:**
- The ServiceAccount is only created when `serviceAccount.create` is `true`.
- ServiceAccount annotations are useful for cloud provider integrations, such as AWS IAM Roles for Service Accounts (IRSA): `eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-role`.

#### templates/NOTES.txt

The `NOTES.txt` file is rendered and displayed to the user after `helm install` or `helm upgrade`. It typically provides instructions on how to access the deployed application.

```
Thank you for installing {{ include "flask-app.fullname" . }}!

Your release is named: {{ .Release.Name }}
Chart version: {{ .Chart.Version }}
App version: {{ .Chart.AppVersion }}

To get the application URL, run these commands:

{{- if .Values.ingress.enabled }}
  The application is accessible via Ingress:
  {{- range $host := .Values.ingress.hosts }}
  {{- range .paths }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ $host.host }}{{ .path }}
  {{- end }}
  {{- end }}

{{- else if contains "NodePort" .Values.service.type }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "flask-app.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo "Application URL: http://$NODE_IP:$NODE_PORT"

{{- else if contains "LoadBalancer" .Values.service.type }}
  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
  You can watch the status with:
    kubectl get --namespace {{ .Release.Namespace }} svc -w {{ include "flask-app.fullname" . }}

  export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "flask-app.fullname" . }} --template "{{"{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}}")
  echo "Application URL: http://$SERVICE_IP:{{ .Values.service.port }}"

{{- else if contains "ClusterIP" .Values.service.type }}
  The application is accessible within the cluster. To access it locally, run:
    export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "{{ include "flask-app.selectorLabels" . | replace ": " "=" | replace "\n" "," }}" -o jsonpath="{.items[0].metadata.name}")
    export CONTAINER_PORT=$(kubectl get pod --namespace {{ .Release.Namespace }} $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
    echo "Visit http://127.0.0.1:8080 to access the application"
    kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:$CONTAINER_PORT
{{- end }}
```

---

## 5. Helm Template Functions and Syntax

### Go Template Basics

Helm templates use Go's `text/template` package with additional functions from the Sprig library and some Helm-specific additions.

**Delimiters:**
- `{{ }}` -- Template action delimiters
- `{{- }}` -- Trim leading whitespace
- `{{ -}}` -- Trim trailing whitespace
- `{{- -}}` -- Trim whitespace on both sides
- `{{/* comment */}}` -- Template comment (not rendered in output)

**Accessing Data:**

The template has access to several built-in objects:

```yaml
# Values from values.yaml
{{ .Values.replicaCount }}
{{ .Values.image.repository }}

# Release information
{{ .Release.Name }}          # The release name (e.g., "my-app")
{{ .Release.Namespace }}     # The namespace being deployed to
{{ .Release.IsUpgrade }}     # True if this is an upgrade
{{ .Release.IsInstall }}     # True if this is an install
{{ .Release.Revision }}      # The revision number (starts at 1)
{{ .Release.Service }}       # Always "Helm"

# Chart metadata from Chart.yaml
{{ .Chart.Name }}            # The chart name
{{ .Chart.Version }}         # The chart version
{{ .Chart.AppVersion }}      # The application version

# Template information
{{ .Template.Name }}         # The path of the current template file
{{ .Template.BasePath }}     # The base path of templates directory

# Kubernetes capabilities
{{ .Capabilities.KubeVersion }}          # The Kubernetes version
{{ .Capabilities.APIVersions }}          # List of available API versions
{{ .Capabilities.APIVersions.Has "apps/v1" }}  # Check if an API version exists
```

### Built-in Functions

Helm includes all Sprig template functions plus some of its own.

#### String Functions

```yaml
# default - Provide a default value if the input is empty
{{ .Values.image.tag | default "latest" }}

# quote - Wrap in double quotes
{{ .Values.env.LOG_LEVEL | quote }}
# Output: "INFO"

# squote - Wrap in single quotes
{{ .Values.env.LOG_LEVEL | squote }}
# Output: 'INFO'

# upper - Convert to uppercase
{{ .Values.env.LOG_LEVEL | upper }}
# Output: INFO

# lower - Convert to lowercase
{{ "HELLO" | lower }}
# Output: hello

# title - Title case
{{ "hello world" | title }}
# Output: Hello World

# trim - Remove leading and trailing whitespace
{{ "  hello  " | trim }}
# Output: hello

# trimSuffix - Remove a suffix
{{ "hello-" | trimSuffix "-" }}
# Output: hello

# trimPrefix - Remove a prefix
{{ "-hello" | trimPrefix "-" }}
# Output: hello

# trunc - Truncate to a given length
{{ "a very long string" | trunc 10 }}
# Output: a very lon

# replace - Replace occurrences
{{ "hello world" | replace " " "-" }}
# Output: hello-world

# contains - Check if a string contains another
{{ if contains "NodePort" .Values.service.type }}...{{ end }}

# hasPrefix / hasSuffix
{{ if hasPrefix "https" .Values.url }}...{{ end }}

# indent - Indent every line by N spaces
{{ .Values.config | indent 4 }}

# nindent - Add a newline, then indent every line by N spaces
{{ toYaml .Values.resources | nindent 12 }}

# printf - Formatted output
{{ printf "%s-%s" .Release.Name .Chart.Name }}

# sha256sum - Generate SHA256 hash (useful for config checksums)
{{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}

# b64enc / b64dec - Base64 encode/decode
{{ "secret" | b64enc }}
# Output: c2VjcmV0
```

#### Type Conversion Functions

```yaml
# int / int64 / float64
{{ int "42" }}

# toString
{{ toString 42 }}

# toJson / mustToJson
{{ .Values.config | toJson }}

# toYaml / mustToYaml
{{ toYaml .Values.resources }}

# toPrettyJson
{{ .Values.config | toPrettyJson }}
```

#### List Functions

```yaml
# list - Create a list
{{ $myList := list "a" "b" "c" }}

# first / last / rest / initial
{{ first (list 1 2 3) }}    # 1
{{ last (list 1 2 3) }}     # 3

# has - Check if a list contains a value
{{ if has "nginx" .Values.tags }}...{{ end }}

# join - Join list elements with a separator
{{ list "a" "b" "c" | join "," }}
# Output: a,b,c

# append / prepend
{{ $myList := append (list "a" "b") "c" }}

# uniq - Remove duplicates
{{ list "a" "b" "a" | uniq }}

# sortAlpha - Sort alphabetically
{{ list "c" "a" "b" | sortAlpha }}
```

#### Dictionary (Map) Functions

```yaml
# dict - Create a dictionary
{{ $myDict := dict "key1" "value1" "key2" "value2" }}

# get - Get a value from a dict
{{ get $myDict "key1" }}

# set - Set a value in a dict (modifies in place)
{{ $_ := set $myDict "key3" "value3" }}

# hasKey - Check if a key exists
{{ if hasKey .Values "ingress" }}...{{ end }}

# keys / values
{{ keys $myDict }}

# merge / mustMerge - Merge dictionaries
{{ merge $dict1 $dict2 }}

# pick / omit - Select or exclude keys
{{ pick $myDict "key1" "key2" }}
{{ omit $myDict "key3" }}
```

### Flow Control

#### if / else / else if

```yaml
# Basic if
{{- if .Values.ingress.enabled }}
  # ... render ingress resource
{{- end }}

# if / else
{{- if .Values.autoscaling.enabled }}
  # HPA manages replicas
{{- else }}
replicas: {{ .Values.replicaCount }}
{{- end }}

# if / else if / else
{{- if eq .Values.service.type "LoadBalancer" }}
  # LoadBalancer specific config
{{- else if eq .Values.service.type "NodePort" }}
  # NodePort specific config
{{- else }}
  # ClusterIP (default)
{{- end }}

# Comparison operators (these are functions, not operators)
{{ if eq .Values.env "production" }}    # Equal
{{ if ne .Values.env "development" }}   # Not equal
{{ if lt .Values.replicas 3 }}          # Less than
{{ if le .Values.replicas 3 }}          # Less than or equal
{{ if gt .Values.replicas 1 }}          # Greater than
{{ if ge .Values.replicas 1 }}          # Greater than or equal

# Logical operators
{{ if and .Values.ingress.enabled .Values.ingress.tls }}
{{ if or .Values.nodeSelector .Values.affinity }}
{{ if not .Values.autoscaling.enabled }}
```

**What counts as "false" in Helm templates:**
- Boolean `false`
- Numeric `0`
- Empty string `""`
- `nil` (null)
- Empty collection (map, list, etc.)

#### with

The `with` action sets the scope (`.`) to a specific value. If the value is empty/nil, the entire block is skipped.

```yaml
# Without 'with' -- verbose
{{- if .Values.nodeSelector }}
nodeSelector:
  {{- toYaml .Values.nodeSelector | nindent 8 }}
{{- end }}

# With 'with' -- cleaner, and also changes scope
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

Inside a `with` block, `.` refers to the `with` value. To access the root scope, use `$`:

```yaml
{{- with .Values.ingress.hosts }}
  {{- range . }}
    host: {{ .host }}
    service: {{ $.Release.Name }}  # Use $ to access root scope
  {{- end }}
{{- end }}
```

#### range

The `range` action iterates over lists and maps.

```yaml
# Iterate over a list
env:
{{- range .Values.extraEnv }}
  - name: {{ .name }}
    value: {{ .value | quote }}
{{- end }}

# Iterate over a map with key-value pairs
{{- range $key, $value := .Values.env }}
  - name: {{ $key }}
    value: {{ $value | quote }}
{{- end }}

# Iterate over a list with index
{{- range $index, $host := .Values.ingress.hosts }}
  # Host {{ $index }}: {{ $host.host }}
{{- end }}
```

### Variables

Variables in Go templates are prefixed with `$`:

```yaml
# Define a variable
{{- $fullName := include "flask-app.fullname" . -}}
{{- $svcPort := .Values.service.port -}}

# Use the variable
name: {{ $fullName }}
port: {{ $svcPort }}

# The root scope is always available as $
# This is especially useful inside range and with blocks
{{- range .Values.ingress.hosts }}
  serviceName: {{ $.Release.Name }}  # $ = root scope
{{- end }}
```

### Named Templates and include

Named templates are defined in `_helpers.tpl` (or any file starting with `_`):

```yaml
# Define a named template
{{- define "flask-app.labels" -}}
app: {{ .Chart.Name }}
release: {{ .Release.Name }}
{{- end }}

# Include a named template (preferred over 'template')
labels:
  {{- include "flask-app.labels" . | nindent 4 }}
```

**`include` vs `template`:**
- `include` returns a string that can be piped to other functions (like `nindent`)
- `template` inserts the result directly and cannot be piped
- Always prefer `include` over `template`

```yaml
# include allows piping -- CORRECT
labels:
  {{- include "flask-app.labels" . | nindent 4 }}

# template does not allow piping -- AVOID
labels:
  {{ template "flask-app.labels" . }}
```

### Required Values

Use the `required` function to fail with a clear error if a critical value is missing:

```yaml
image:
  repository: {{ required "image.repository is required" .Values.image.repository }}
  tag: {{ required "image.tag must be set" .Values.image.tag }}
```

If `image.repository` is not set, Helm will fail with:

```
Error: execution error at (flask-app/templates/deployment.yaml:20:20):
image.repository is required
```

### toYaml and nindent

These two functions are used together constantly in Helm templates:

```yaml
# toYaml converts a Go data structure to a YAML string
# nindent N adds a newline and indents every line by N spaces

# Example: rendering resources from values
resources:
  {{- toYaml .Values.resources | nindent 2 }}

# This renders as:
resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

**Getting indentation right is critical.** The `nindent` value must match the indentation level where the content should appear. Count the spaces from the left margin to where the content should start.

### The lookup Function

The `lookup` function queries live Kubernetes resources during template rendering. It only works during `helm install` and `helm upgrade` (not during `helm template` or `--dry-run`).

```yaml
# lookup APIVERSION KIND NAMESPACE NAME
# Returns a single resource or a list

# Look up a specific Secret
{{- $secret := lookup "v1" "Secret" .Release.Namespace "my-secret" -}}
{{- if $secret }}
  # Secret exists, use its data
  password: {{ index $secret.data "password" }}
{{- else }}
  # Secret doesn't exist, generate a new password
  password: {{ randAlphaNum 16 | b64enc }}
{{- end }}

# Look up all ConfigMaps in the namespace
{{- $configmaps := lookup "v1" "ConfigMap" .Release.Namespace "" -}}

# Look up across all namespaces (empty string for namespace)
{{- $nodes := lookup "v1" "Node" "" "" -}}
```

---

## 6. Testing and Validating Charts

### helm template -- Render Templates Locally

The `helm template` command renders your chart templates locally without connecting to a Kubernetes cluster. This is invaluable for development and debugging.

```bash
# Render all templates with default values
helm template my-release ./flask-app

# Render with custom values
helm template my-release ./flask-app -f values-prod.yaml

# Render a specific template
helm template my-release ./flask-app -s templates/deployment.yaml

# Render with overridden values
helm template my-release ./flask-app --set replicaCount=3

# Include CRDs in the output
helm template my-release ./flask-app --include-crds

# Render for a specific namespace
helm template my-release ./flask-app --namespace production

# Output to a file for review
helm template my-release ./flask-app > rendered-manifests.yaml

# Validate the rendered output against Kubernetes schemas
helm template my-release ./flask-app | kubectl apply --dry-run=client -f -
```

### helm lint -- Validate Chart Structure

`helm lint` examines a chart for possible issues and reports warnings and errors:

```bash
# Lint with default values
helm lint ./flask-app

# Lint with custom values
helm lint ./flask-app -f values-prod.yaml

# Lint with strict mode (treats warnings as errors)
helm lint ./flask-app --strict

# Lint with debug output
helm lint ./flask-app --debug
```

Example output:

```
==> Linting ./flask-app
[INFO] Chart.yaml: icon is recommended
[WARNING] templates/ingress.yaml: object name does not conform to Kubernetes naming requirements
[ERROR] templates/deployment.yaml: unable to parse YAML

Error: 1 chart(s) linted, 1 chart(s) failed
```

**Common issues `helm lint` catches:**
- Invalid YAML syntax
- Missing required fields in Chart.yaml
- Template rendering errors
- Naming convention violations

### helm install --dry-run --debug

This performs a server-side dry run, which means Helm talks to the Kubernetes API server to validate the rendered manifests, but does not actually create any resources:

```bash
helm install my-release ./flask-app --dry-run --debug
```

The output includes:
- Computed values (all defaults merged with overrides)
- Every rendered template
- The rendered NOTES.txt
- Any warnings or errors

This is more thorough than `helm template` because the Kubernetes API server validates the manifests (checking for valid API versions, required fields, etc.).

```bash
# Dry run with custom values
helm install my-release ./flask-app --dry-run --debug -f values-prod.yaml

# Dry run an upgrade
helm upgrade my-release ./flask-app --dry-run --debug
```

### Chart Testing with helm test

Helm supports running test pods as part of a release. Tests are defined as pod specs in the `templates/tests/` directory with the `helm.sh/hook: test` annotation.

#### Writing Test Pods

Create a file at `templates/tests/test-connection.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "flask-app.fullname" . }}-test-connection"
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  containers:
    - name: wget
      image: busybox:1.36
      command: ['wget']
      args:
        - '--spider'
        - '--timeout=5'
        - '{{ include "flask-app.fullname" . }}:{{ .Values.service.port }}/health'
  restartPolicy: Never
```

Create a more comprehensive test at `templates/tests/test-api.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "flask-app.fullname" . }}-test-api"
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  containers:
    - name: curl
      image: curlimages/curl:8.5.0
      command:
        - /bin/sh
        - -c
        - |
          echo "Testing health endpoint..."
          HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://{{ include "flask-app.fullname" . }}:{{ .Values.service.port }}/health)
          if [ "$HTTP_CODE" -ne 200 ]; then
            echo "FAIL: Health endpoint returned $HTTP_CODE"
            exit 1
          fi
          echo "PASS: Health endpoint returned 200"

          echo "Testing root endpoint..."
          HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://{{ include "flask-app.fullname" . }}:{{ .Values.service.port }}/)
          if [ "$HTTP_CODE" -ne 200 ]; then
            echo "FAIL: Root endpoint returned $HTTP_CODE"
            exit 1
          fi
          echo "PASS: Root endpoint returned 200"

          echo "All tests passed!"
  restartPolicy: Never
```

#### Running Tests

```bash
# Run tests for a release
helm test my-release

# Run tests with timeout
helm test my-release --timeout 5m

# Run tests with logs output
helm test my-release --logs
```

Output:

```
NAME: my-release
LAST DEPLOYED: Mon Jan 15 10:30:00 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE:     my-release-flask-app-test-connection
Last Started:   Mon Jan 15 10:31:00 2024
Last Completed: Mon Jan 15 10:31:05 2024
Phase:          Succeeded
TEST SUITE:     my-release-flask-app-test-api
Last Started:   Mon Jan 15 10:31:06 2024
Last Completed: Mon Jan 15 10:31:12 2024
Phase:          Succeeded
```

---

## 7. Advanced Helm Usage

### Multiple Environments

One of Helm's greatest strengths is deploying the same chart across different environments with different configurations. The recommended approach is to use separate values files for each environment.

#### values-dev.yaml

```yaml
# values-dev.yaml -- Development environment overrides

replicaCount: 1

image:
  repository: myregistry.example.com/flask-app
  tag: "dev-latest"
  pullPolicy: Always

service:
  type: NodePort
  port: 80
  targetPort: 5000
  nodePort: 30080

ingress:
  enabled: false

resources:
  limits:
    cpu: 250m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

autoscaling:
  enabled: false

env:
  FLASK_ENV: "development"
  FLASK_DEBUG: "1"
  LOG_LEVEL: "DEBUG"
  PORT: "5000"

configMap:
  enabled: true
  data:
    APP_NAME: "Flask App (DEV)"
    WORKERS: "1"
    TIMEOUT: "300"

# Disable probes in development for faster iteration
livenessProbe: {}
readinessProbe: {}
startupProbe: {}
```

#### values-staging.yaml

```yaml
# values-staging.yaml -- Staging environment overrides

replicaCount: 2

image:
  repository: myregistry.example.com/flask-app
  tag: "staging-latest"
  pullPolicy: Always

service:
  type: ClusterIP
  port: 80
  targetPort: 5000

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
  hosts:
    - host: flask-app.staging.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: flask-app-staging-tls
      hosts:
        - flask-app.staging.example.com

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false

env:
  FLASK_ENV: "staging"
  FLASK_DEBUG: "0"
  LOG_LEVEL: "INFO"
  PORT: "5000"

configMap:
  enabled: true
  data:
    APP_NAME: "Flask App (STAGING)"
    WORKERS: "2"
    TIMEOUT: "120"
```

#### values-prod.yaml

```yaml
# values-prod.yaml -- Production environment overrides

replicaCount: 3

image:
  repository: myregistry.example.com/flask-app
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 5000

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/rate-limit-connections: "10"
  hosts:
    - host: flask-app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: flask-app-prod-tls
      hosts:
        - flask-app.example.com

resources:
  limits:
    cpu: 1000m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 75

podSecurityContext:
  fsGroup: 1000
  runAsNonRoot: true

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000

nodeSelector:
  kubernetes.io/os: linux

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                  - flask-app
          topologyKey: kubernetes.io/hostname

env:
  FLASK_ENV: "production"
  FLASK_DEBUG: "0"
  LOG_LEVEL: "WARNING"
  PORT: "5000"

envFromSecrets:
  - name: DATABASE_URL
    secretName: flask-app-prod-secrets
    secretKey: database-url
  - name: SECRET_KEY
    secretName: flask-app-prod-secrets
    secretKey: secret-key

configMap:
  enabled: true
  data:
    APP_NAME: "Flask App"
    WORKERS: "4"
    TIMEOUT: "120"
```

#### Deploying to Each Environment

```bash
# Deploy to development
helm upgrade --install flask-app-dev ./flask-app \
  -f values-dev.yaml \
  --namespace development \
  --create-namespace

# Deploy to staging
helm upgrade --install flask-app-staging ./flask-app \
  -f values-staging.yaml \
  --namespace staging \
  --create-namespace

# Deploy to production
helm upgrade --install flask-app-prod ./flask-app \
  -f values-prod.yaml \
  --namespace production \
  --create-namespace
```

#### Merging Multiple Values Files

Helm merges values files in order, with later files overriding earlier ones:

```bash
# Base values + environment-specific overrides
helm upgrade --install flask-app ./flask-app \
  -f values.yaml \
  -f values-prod.yaml \
  --set image.tag="1.2.3"
```

Priority order (highest to lowest):
1. `--set` flags
2. Last `-f` file (`values-prod.yaml`)
3. First `-f` file (`values.yaml`)
4. Chart's default `values.yaml`

### Helm Hooks

Hooks allow you to execute operations at specific points in a release's lifecycle. They are regular Kubernetes resources (usually Jobs or Pods) with special annotations.

#### Available Hook Types

| Hook | Description | Use Case |
|---|---|---|
| `pre-install` | Runs before any release resources are installed | Set up prerequisites, validate environment |
| `post-install` | Runs after all release resources are installed | Load seed data, send notifications |
| `pre-upgrade` | Runs before any release resources are upgraded | Database backup before schema migration |
| `post-upgrade` | Runs after all release resources are upgraded | Run database migrations, cache warming |
| `pre-delete` | Runs before any release resources are deleted | Backup data, graceful shutdown tasks |
| `post-delete` | Runs after all release resources are deleted | Cleanup external resources |
| `pre-rollback` | Runs before a rollback operation | Backup before rollback |
| `post-rollback` | Runs after a rollback operation | Restore configuration |
| `test` | Runs when `helm test` is executed | Validate deployment |

#### Hook Delete Policies

| Policy | Description |
|---|---|
| `before-hook-creation` | Delete previous hook resources before new hook runs (default) |
| `hook-succeeded` | Delete hook resources after the hook succeeds |
| `hook-failed` | Delete hook resources if the hook fails |

#### Hook Weight

Hooks can be ordered using weights. Lower weights run first:

```yaml
annotations:
  "helm.sh/hook-weight": "-5"  # Runs before weight "0"
```

#### Example: Database Migration Hook

```yaml
# templates/hooks/db-migrate.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "flask-app.fullname" . }}-db-migrate
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
  annotations:
    # Run this job before upgrading the deployment
    "helm.sh/hook": pre-upgrade,pre-install
    # Run before other hooks
    "helm.sh/hook-weight": "-5"
    # Clean up the job after it succeeds
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 3
  template:
    metadata:
      labels:
        {{- include "flask-app.selectorLabels" . | nindent 8 }}
    spec:
      restartPolicy: Never
      containers:
        - name: db-migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          command: ["python", "manage.py", "db", "upgrade"]
          env:
            {{- range .Values.envFromSecrets }}
            - name: {{ .name }}
              valueFrom:
                secretKeyRef:
                  name: {{ .secretName }}
                  key: {{ .secretKey }}
            {{- end }}
```

#### Example: Post-Install Notification Hook

```yaml
# templates/hooks/post-install-notify.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "flask-app.fullname" . }}-notify
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: notify
          image: curlimages/curl:8.5.0
          command:
            - /bin/sh
            - -c
            - |
              curl -X POST https://hooks.slack.com/services/XXX/YYY/ZZZ \
                -H 'Content-type: application/json' \
                -d '{"text":"Release {{ .Release.Name }} ({{ .Chart.AppVersion }}) deployed to {{ .Release.Namespace }}"}'
```

#### Example: Pre-Delete Backup Hook

```yaml
# templates/hooks/pre-delete-backup.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "flask-app.fullname" . }}-pre-delete-backup
  annotations:
    "helm.sh/hook": pre-delete
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: backup
          image: postgres:16
          command:
            - /bin/sh
            - -c
            - |
              pg_dump $DATABASE_URL > /backup/dump-$(date +%Y%m%d-%H%M%S).sql
              echo "Backup completed before deletion"
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: flask-app-prod-secrets
                  key: database-url
          volumeMounts:
            - name: backup-volume
              mountPath: /backup
      volumes:
        - name: backup-volume
          persistentVolumeClaim:
            claimName: backup-pvc
```

### Dependencies

Charts can depend on other charts. For example, your Flask app might need Redis for caching and PostgreSQL for data storage.

#### Declaring Dependencies in Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: flask-app
version: 0.1.0
appVersion: "1.0.0"

dependencies:
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
    tags:
      - backend
  - name: postgresql
    version: "14.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
    tags:
      - backend
```

**Dependency fields:**

| Field | Description |
|---|---|
| `name` | The name of the chart |
| `version` | SemVer range (e.g., `"18.x.x"`, `">=14.0.0"`, `"~14.3.0"`) |
| `repository` | The URL of the chart repository |
| `condition` | A values path that enables/disables this dependency |
| `tags` | Group dependencies for bulk enable/disable |
| `alias` | An alternative name for the dependency |
| `import-values` | Import values from the child chart |

#### Managing Dependencies

```bash
# Download and lock dependencies (creates/updates Chart.lock and charts/ directory)
helm dependency update ./flask-app

# Rebuild the charts/ directory from Chart.lock (without updating versions)
helm dependency build ./flask-app

# List dependencies and their status
helm dependency list ./flask-app
```

Output of `helm dependency list`:

```
NAME            VERSION   REPOSITORY                              STATUS
redis           18.x.x    https://charts.bitnami.com/bitnami      ok
postgresql      14.x.x    https://charts.bitnami.com/bitnami      ok
```

#### Configuring Dependencies via values.yaml

Sub-chart values are nested under the dependency name:

```yaml
# values.yaml

# Enable/disable dependencies
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true
    password: "my-redis-password"
  master:
    resources:
      requests:
        cpu: 100m
        memory: 128Mi

postgresql:
  enabled: true
  auth:
    username: flaskapp
    password: "my-db-password"
    database: flaskapp
  primary:
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
```

### Packaging and Sharing Charts

#### Packaging a Chart

```bash
# Package the chart into a .tgz archive
helm package ./flask-app

# Output: flask-app-0.1.0.tgz

# Package with a specific version override
helm package ./flask-app --version 1.0.0

# Package with a specific app version
helm package ./flask-app --app-version 2.0.0

# Package to a specific destination directory
helm package ./flask-app --destination ./charts-repo/

# Package with signing (GPG)
helm package ./flask-app --sign --key "DevOps Team" --keyring ~/.gnupg/secring.gpg
```

#### Hosting a Chart Repository

A chart repository is simply an HTTP server that serves an `index.yaml` file and chart `.tgz` archives.

**Generate the repository index:**

```bash
# Create a directory for your chart repository
mkdir -p ./charts-repo

# Copy packaged charts into the directory
cp flask-app-0.1.0.tgz ./charts-repo/

# Generate the index file
helm repo index ./charts-repo --url https://charts.example.com

# Merge new charts into an existing index
helm repo index ./charts-repo --url https://charts.example.com --merge ./charts-repo/index.yaml
```

The generated `index.yaml` looks like:

```yaml
apiVersion: v1
entries:
  flask-app:
    - apiVersion: v2
      appVersion: "1.0.0"
      created: "2024-01-15T10:30:00Z"
      description: A Helm chart for deploying a Python Flask web application
      digest: sha256:abc123...
      name: flask-app
      type: application
      urls:
        - https://charts.example.com/flask-app-0.1.0.tgz
      version: 0.1.0
generated: "2024-01-15T10:30:00Z"
```

**Using OCI Registries:**

Helm 3 supports pushing charts to OCI-compliant registries (Docker Hub, AWS ECR, Google Artifact Registry, etc.):

```bash
# Log in to the registry
helm registry login registry.example.com

# Push a chart to an OCI registry
helm push flask-app-0.1.0.tgz oci://registry.example.com/charts

# Pull a chart from an OCI registry
helm pull oci://registry.example.com/charts/flask-app --version 0.1.0

# Install directly from an OCI registry
helm install my-release oci://registry.example.com/charts/flask-app --version 0.1.0
```

---

## 8. Helm in CI/CD Pipelines

### How Helm Fits in CI/CD

Helm is commonly used in the deployment stage of CI/CD pipelines. A typical workflow looks like:

1. **Build** -- Build application code, run tests
2. **Containerize** -- Build a Docker image, push to registry
3. **Deploy** -- Use Helm to deploy/upgrade the application on Kubernetes

### Installing Helm on Jenkins Agents

If your Jenkins agents do not have Helm pre-installed:

```groovy
// In a Jenkinsfile stage
stage('Setup Tools') {
    steps {
        sh '''
            # Install Helm if not present
            if ! command -v helm &> /dev/null; then
                curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
            fi
            helm version
        '''
    }
}
```

Alternatively, use a Docker-based agent that includes Helm:

```groovy
pipeline {
    agent {
        docker {
            image 'alpine/helm:3.16.4'
            args '--entrypoint=""'
        }
    }
    // ...
}
```

### kubeconfig Management

For Jenkins to deploy to Kubernetes, it needs a valid `kubeconfig`:

```groovy
// Using Jenkins credentials to inject kubeconfig
stage('Deploy') {
    steps {
        withCredentials([file(credentialsId: 'kubeconfig-prod', variable: 'KUBECONFIG')]) {
            sh '''
                export KUBECONFIG=$KUBECONFIG
                kubectl cluster-info
                helm list -A
            '''
        }
    }
}
```

### Example Jenkins Pipeline Using Helm

```groovy
pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'myregistry.example.com'
        CHART_DIR = 'helm/flask-app'
        APP_NAME = 'flask-app'
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['development', 'staging', 'production'],
            description: 'Target deployment environment'
        )
        string(
            name: 'IMAGE_TAG',
            defaultValue: '',
            description: 'Docker image tag to deploy (leave empty for BUILD_NUMBER)'
        )
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                script {
                    def imageTag = params.IMAGE_TAG ?: env.BUILD_NUMBER
                    env.DEPLOY_TAG = imageTag

                    sh """
                        docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${imageTag} .
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}:${imageTag}
                    """
                }
            }
        }

        stage('Lint Helm Chart') {
            steps {
                sh """
                    helm lint ${CHART_DIR} \
                        -f ${CHART_DIR}/values-${params.ENVIRONMENT}.yaml \
                        --strict
                """
            }
        }

        stage('Dry Run') {
            steps {
                withCredentials([file(credentialsId: "kubeconfig-${params.ENVIRONMENT}", variable: 'KUBECONFIG')]) {
                    sh """
                        export KUBECONFIG=\$KUBECONFIG

                        helm upgrade --install ${APP_NAME} ${CHART_DIR} \
                            -f ${CHART_DIR}/values-${params.ENVIRONMENT}.yaml \
                            --set image.tag=${env.DEPLOY_TAG} \
                            --namespace ${params.ENVIRONMENT} \
                            --dry-run --debug
                    """
                }
            }
        }

        stage('Approval') {
            when {
                expression { params.ENVIRONMENT == 'production' }
            }
            steps {
                input message: "Deploy version ${env.DEPLOY_TAG} to production?",
                      ok: 'Deploy'
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([file(credentialsId: "kubeconfig-${params.ENVIRONMENT}", variable: 'KUBECONFIG')]) {
                    sh """
                        export KUBECONFIG=\$KUBECONFIG

                        helm upgrade --install ${APP_NAME} ${CHART_DIR} \
                            -f ${CHART_DIR}/values-${params.ENVIRONMENT}.yaml \
                            --set image.tag=${env.DEPLOY_TAG} \
                            --namespace ${params.ENVIRONMENT} \
                            --create-namespace \
                            --wait \
                            --timeout 5m
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([file(credentialsId: "kubeconfig-${params.ENVIRONMENT}", variable: 'KUBECONFIG')]) {
                    sh """
                        export KUBECONFIG=\$KUBECONFIG

                        # Check release status
                        helm status ${APP_NAME} --namespace ${params.ENVIRONMENT}

                        # Run Helm tests
                        helm test ${APP_NAME} --namespace ${params.ENVIRONMENT} --timeout 2m

                        # Verify pods are running
                        kubectl get pods --namespace ${params.ENVIRONMENT} -l app.kubernetes.io/name=${APP_NAME}
                    """
                }
            }
        }

        stage('Rollback on Failure') {
            when {
                expression { currentBuild.result == 'FAILURE' }
            }
            steps {
                withCredentials([file(credentialsId: "kubeconfig-${params.ENVIRONMENT}", variable: 'KUBECONFIG')]) {
                    sh """
                        export KUBECONFIG=\$KUBECONFIG
                        echo "Deployment failed. Rolling back..."
                        helm rollback ${APP_NAME} --namespace ${params.ENVIRONMENT}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Successfully deployed ${APP_NAME}:${env.DEPLOY_TAG} to ${params.ENVIRONMENT}"
        }
        failure {
            echo "Deployment of ${APP_NAME}:${env.DEPLOY_TAG} to ${params.ENVIRONMENT} failed"
        }
    }
}
```

### Automated Version Bumping

A script to automatically bump the chart version:

```bash
#!/bin/bash
# bump-chart-version.sh
# Usage: ./bump-chart-version.sh [major|minor|patch]

CHART_FILE="Chart.yaml"
BUMP_TYPE="${1:-patch}"

# Extract current version
CURRENT_VERSION=$(grep '^version:' "$CHART_FILE" | awk '{print $2}')

# Split into major.minor.patch
IFS='.' read -r MAJOR MINOR PATCH <<< "$CURRENT_VERSION"

case "$BUMP_TYPE" in
    major)
        MAJOR=$((MAJOR + 1))
        MINOR=0
        PATCH=0
        ;;
    minor)
        MINOR=$((MINOR + 1))
        PATCH=0
        ;;
    patch)
        PATCH=$((PATCH + 1))
        ;;
    *)
        echo "Usage: $0 [major|minor|patch]"
        exit 1
        ;;
esac

NEW_VERSION="${MAJOR}.${MINOR}.${PATCH}"

# Update Chart.yaml
sed -i "s/^version: .*/version: ${NEW_VERSION}/" "$CHART_FILE"

echo "Bumped chart version: ${CURRENT_VERSION} -> ${NEW_VERSION}"
```

---

## 9. Helm Best Practices

### Chart Versioning

- Follow **Semantic Versioning** (SemVer 2.0.0) for chart versions
- The chart `version` and `appVersion` are independent -- bump them separately
- Increment `version` whenever anything in the chart changes (templates, values, dependencies)
- Increment `appVersion` when the underlying application version changes
- Use pre-release versions during development: `0.1.0-alpha.1`, `0.1.0-beta.1`

### Values File Organization

- **Keep `values.yaml` well-documented.** Add comments explaining every value, its purpose, and valid options.
- **Use sensible defaults.** The chart should be installable with zero overrides for a basic development setup.
- **Group related values.** For example, all image-related values under `image:`, all ingress values under `ingress:`.
- **Use `enabled` flags** for optional features (ingress, autoscaling, etc.) rather than relying on empty values.
- **Never put secrets in `values.yaml`.** Use `envFromSecrets` or external secret management.

### Template Reusability

- **Use `_helpers.tpl`** for all repeated logic (names, labels, selectors).
- **Prefer `include` over `template`.** `include` can be piped to functions like `nindent`.
- **Use `with` blocks** instead of `if` + `toYaml` for optional sections.
- **Use `required`** for values that must be provided by the user.
- **Use the `default` function** for values that have sensible fallbacks.

### Security Best Practices

- **Run as non-root.** Set `runAsNonRoot: true` and `runAsUser: 1000` in `securityContext`.
- **Drop all capabilities.** Use `capabilities: { drop: [ALL] }`.
- **Use read-only root filesystem** where possible: `readOnlyRootFilesystem: true`.
- **Set resource limits.** Always define `resources.requests` and `resources.limits`.
- **Use ServiceAccounts** with minimal permissions.
- **Never hardcode secrets.** Use Kubernetes Secrets referenced via `secretKeyRef`.
- **Use `imagePullPolicy: IfNotPresent`** for tagged images, `Always` for mutable tags like `latest`.
- **Sign your charts** with GPG when distributing them.

### Documentation

- Include a thorough `NOTES.txt` that shows users how to access the deployed application.
- Comment every value in `values.yaml` with `# --` prefix (this enables automatic documentation generation with tools like `helm-docs`).
- Include a `README.md` in the chart root documenting prerequisites, configuration options, and usage examples.

### General Guidelines

- **Always use `helm upgrade --install`** instead of separate install/upgrade commands. This makes scripts idempotent.
- **Always use `--wait`** in CI/CD to ensure the deployment is healthy before proceeding.
- **Use `--atomic`** for production deployments -- this automatically rolls back on failure.
- **Pin chart versions** in CI/CD pipelines (do not use `latest` or ranges).
- **Use namespaces** to isolate environments.
- **Test charts** with `helm lint`, `helm template`, and `helm test` in your CI pipeline.

---

## 10. Helm Commands Reference

### Chart Management

| Command | Description |
|---|---|
| `helm create <name>` | Create a new chart with the given name |
| `helm package <chart-path>` | Package a chart directory into a `.tgz` archive |
| `helm lint <chart-path>` | Examine a chart for possible issues |
| `helm template <name> <chart>` | Render chart templates locally |
| `helm show all <chart>` | Show all information about a chart |
| `helm show chart <chart>` | Show the chart's `Chart.yaml` |
| `helm show values <chart>` | Show the chart's default `values.yaml` |
| `helm show readme <chart>` | Show the chart's README |
| `helm pull <chart>` | Download a chart from a repository |
| `helm push <chart> <remote>` | Push a chart to a remote registry (OCI) |
| `helm dependency update <chart>` | Update chart dependencies |
| `helm dependency build <chart>` | Rebuild the `charts/` directory from `Chart.lock` |
| `helm dependency list <chart>` | List dependencies for a chart |

### Release Management

| Command | Description |
|---|---|
| `helm install <name> <chart>` | Install a chart as a new release |
| `helm upgrade <name> <chart>` | Upgrade an existing release |
| `helm upgrade --install <name> <chart>` | Install or upgrade a release (idempotent) |
| `helm rollback <name> [revision]` | Roll back a release to a previous revision |
| `helm uninstall <name>` | Uninstall a release |
| `helm list` | List releases in the current namespace |
| `helm list -A` | List releases across all namespaces |
| `helm status <name>` | Show the status of a release |
| `helm history <name>` | Show the release history |
| `helm test <name>` | Run tests for a release |
| `helm get all <name>` | Get all information about a release |
| `helm get values <name>` | Get the values used for a release |
| `helm get manifest <name>` | Get the rendered manifest for a release |
| `helm get notes <name>` | Get the release notes |
| `helm get hooks <name>` | Get the hooks for a release |

### Repository Management

| Command | Description |
|---|---|
| `helm repo add <name> <url>` | Add a chart repository |
| `helm repo list` | List added chart repositories |
| `helm repo update` | Update local cache of repository charts |
| `helm repo remove <name>` | Remove a chart repository |
| `helm repo index <dir>` | Generate an `index.yaml` for a chart repository |
| `helm search repo <keyword>` | Search repositories for charts |
| `helm search hub <keyword>` | Search Artifact Hub for charts |

### Registry Commands (OCI)

| Command | Description |
|---|---|
| `helm registry login <host>` | Log in to an OCI registry |
| `helm registry logout <host>` | Log out from an OCI registry |
| `helm push <chart.tgz> <ref>` | Push a chart to an OCI registry |
| `helm pull <ref>` | Pull a chart from an OCI registry |

### Environment and Debugging

| Command | Description |
|---|---|
| `helm version` | Show the Helm client version |
| `helm env` | Show Helm environment variables |
| `helm plugin list` | List installed Helm plugins |
| `helm plugin install <url>` | Install a Helm plugin |
| `helm plugin uninstall <name>` | Uninstall a Helm plugin |
| `helm completion bash` | Generate bash autocompletion script |
| `helm completion zsh` | Generate zsh autocompletion script |

### Useful Flags (Common Across Commands)

| Flag | Description |
|---|---|
| `--namespace <ns>` | Specify the Kubernetes namespace |
| `--create-namespace` | Create the namespace if it does not exist |
| `-f <file>` | Provide a values file |
| `--set key=value` | Override a specific value |
| `--set-string key=value` | Override a value and force it to be a string |
| `--set-file key=filepath` | Set a value from the contents of a file |
| `--dry-run` | Simulate the operation without making changes |
| `--debug` | Enable verbose output |
| `--wait` | Wait until resources are ready |
| `--timeout <duration>` | Time to wait for operations (e.g., `5m`, `300s`) |
| `--atomic` | Roll back on failure during install/upgrade |
| `--force` | Force resource updates |
| `--output json\|yaml\|table` | Output format |
| `--kube-context <ctx>` | Use a specific kubeconfig context |
| `--kubeconfig <file>` | Path to a kubeconfig file |

---

## 11. Practice Exercises

### Exercise 1: Install and Explore a Chart from a Repository

**Objective:** Get familiar with Helm repositories and installing pre-built charts.

**Steps:**

1. Add the Bitnami repository:
   ```bash
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm repo update
   ```

2. Search for the NGINX chart:
   ```bash
   helm search repo bitnami/nginx
   ```

3. View the default values for the NGINX chart:
   ```bash
   helm show values bitnami/nginx > nginx-defaults.yaml
   ```

4. Open `nginx-defaults.yaml` and review the available configuration options.

5. Install NGINX with custom values:
   ```bash
   helm install my-nginx bitnami/nginx \
     --set replicaCount=2 \
     --set service.type=ClusterIP \
     --namespace exercise-1 \
     --create-namespace
   ```

6. Check the release status:
   ```bash
   helm status my-nginx --namespace exercise-1
   helm list --namespace exercise-1
   ```

7. View the running pods:
   ```bash
   kubectl get pods --namespace exercise-1
   kubectl get svc --namespace exercise-1
   ```

8. Upgrade the release to change the replica count:
   ```bash
   helm upgrade my-nginx bitnami/nginx \
     --set replicaCount=3 \
     --set service.type=ClusterIP \
     --namespace exercise-1
   ```

9. View the release history:
   ```bash
   helm history my-nginx --namespace exercise-1
   ```

10. Roll back to revision 1:
    ```bash
    helm rollback my-nginx 1 --namespace exercise-1
    ```

11. Verify the rollback (should be back to 2 replicas):
    ```bash
    kubectl get pods --namespace exercise-1
    ```

12. Clean up:
    ```bash
    helm uninstall my-nginx --namespace exercise-1
    kubectl delete namespace exercise-1
    ```

---

### Exercise 2: Create Your First Helm Chart

**Objective:** Build a Helm chart from scratch for a simple NGINX-based static website.

**Steps:**

1. Scaffold a new chart:
   ```bash
   helm create my-website
   ```

2. Examine the generated files:
   ```bash
   ls -la my-website/
   ls -la my-website/templates/
   cat my-website/Chart.yaml
   cat my-website/values.yaml
   ```

3. Modify `values.yaml` to use the standard NGINX image:
   ```yaml
   replicaCount: 2

   image:
     repository: nginx
     pullPolicy: IfNotPresent
     tag: "1.25"

   service:
     type: NodePort
     port: 80

   resources:
     limits:
       cpu: 100m
       memory: 128Mi
     requests:
       cpu: 50m
       memory: 64Mi
   ```

4. Lint the chart:
   ```bash
   helm lint ./my-website
   ```

5. Render the templates locally:
   ```bash
   helm template my-site ./my-website
   ```

6. Review the output. Are the replica count, image, and service type correct?

7. Install the chart:
   ```bash
   helm install my-site ./my-website \
     --namespace exercise-2 \
     --create-namespace
   ```

8. Verify the deployment:
   ```bash
   kubectl get all --namespace exercise-2
   ```

9. Clean up:
   ```bash
   helm uninstall my-site --namespace exercise-2
   kubectl delete namespace exercise-2
   rm -rf my-website/
   ```

---

### Exercise 3: Environment-Specific Deployments

**Objective:** Deploy the same chart to multiple environments with different configurations.

**Steps:**

1. Create a chart:
   ```bash
   helm create multi-env-app
   ```

2. Create three values files:

   `values-dev.yaml`:
   ```yaml
   replicaCount: 1
   image:
     repository: nginx
     tag: "1.25"
   service:
     type: NodePort
     port: 80
   resources:
     limits:
       cpu: 100m
       memory: 64Mi
     requests:
       cpu: 50m
       memory: 32Mi
   ```

   `values-staging.yaml`:
   ```yaml
   replicaCount: 2
   image:
     repository: nginx
     tag: "1.25"
   service:
     type: ClusterIP
     port: 80
   resources:
     limits:
       cpu: 250m
       memory: 128Mi
     requests:
       cpu: 100m
       memory: 64Mi
   ```

   `values-prod.yaml`:
   ```yaml
   replicaCount: 3
   image:
     repository: nginx
     tag: "1.25"
   service:
     type: ClusterIP
     port: 80
   resources:
     limits:
       cpu: 500m
       memory: 256Mi
     requests:
       cpu: 250m
       memory: 128Mi
   ```

3. Render each environment's templates and compare the differences:
   ```bash
   helm template app-dev ./multi-env-app -f values-dev.yaml > rendered-dev.yaml
   helm template app-staging ./multi-env-app -f values-staging.yaml > rendered-staging.yaml
   helm template app-prod ./multi-env-app -f values-prod.yaml > rendered-prod.yaml

   diff rendered-dev.yaml rendered-staging.yaml
   diff rendered-staging.yaml rendered-prod.yaml
   ```

4. Deploy to the "dev" environment:
   ```bash
   helm upgrade --install app-dev ./multi-env-app \
     -f values-dev.yaml \
     --namespace dev \
     --create-namespace
   ```

5. Deploy to the "staging" environment:
   ```bash
   helm upgrade --install app-staging ./multi-env-app \
     -f values-staging.yaml \
     --namespace staging \
     --create-namespace
   ```

6. Compare the deployments:
   ```bash
   kubectl get pods --namespace dev
   kubectl get pods --namespace staging
   ```

7. Clean up:
   ```bash
   helm uninstall app-dev --namespace dev
   helm uninstall app-staging --namespace staging
   kubectl delete namespace dev staging
   rm -rf multi-env-app/ values-*.yaml rendered-*.yaml
   ```

---

### Exercise 4: Add a ConfigMap and Custom Template Logic

**Objective:** Practice writing Helm templates with conditionals, loops, and ConfigMaps.

**Steps:**

1. Create a chart:
   ```bash
   helm create configmap-exercise
   ```

2. Add a ConfigMap template at `configmap-exercise/templates/configmap.yaml`:
   ```yaml
   {{- if .Values.config.enabled }}
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: {{ include "configmap-exercise.fullname" . }}-config
     labels:
       {{- include "configmap-exercise.labels" . | nindent 4 }}
   data:
     {{- range $key, $value := .Values.config.data }}
     {{ $key }}: {{ $value | quote }}
     {{- end }}
   {{- end }}
   ```

3. Add config values to `values.yaml`:
   ```yaml
   config:
     enabled: true
     data:
       DATABASE_HOST: "localhost"
       DATABASE_PORT: "5432"
       LOG_LEVEL: "INFO"
       CACHE_TTL: "3600"
   ```

4. Modify the Deployment template to mount the ConfigMap as environment variables. Add this to the container spec in `deployment.yaml`:
   ```yaml
   {{- if .Values.config.enabled }}
   envFrom:
     - configMapRef:
         name: {{ include "configmap-exercise.fullname" . }}-config
   {{- end }}
   ```

5. Render the templates and verify the ConfigMap is correctly generated:
   ```bash
   helm template my-app ./configmap-exercise -s templates/configmap.yaml
   ```

6. Test with config disabled:
   ```bash
   helm template my-app ./configmap-exercise \
     -s templates/configmap.yaml \
     --set config.enabled=false
   ```
   (The output should be empty.)

7. Lint and validate:
   ```bash
   helm lint ./configmap-exercise --strict
   ```

8. Clean up:
   ```bash
   rm -rf configmap-exercise/
   ```

---

### Exercise 5: Chart Packaging and Lifecycle Management

**Objective:** Practice packaging charts, managing releases through their full lifecycle, and running tests.

**Steps:**

1. Create a chart:
   ```bash
   helm create lifecycle-app
   ```

2. Add a test pod at `lifecycle-app/templates/tests/test-connection.yaml`:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: "{{ include "lifecycle-app.fullname" . }}-test-connection"
     labels:
       {{- include "lifecycle-app.labels" . | nindent 4 }}
     annotations:
       "helm.sh/hook": test
       "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
   spec:
     containers:
       - name: wget
         image: busybox:1.36
         command: ['wget']
         args: ['--spider', '--timeout=5', '{{ include "lifecycle-app.fullname" . }}:{{ .Values.service.port }}']
     restartPolicy: Never
   ```

3. Lint the chart:
   ```bash
   helm lint ./lifecycle-app --strict
   ```

4. Package the chart:
   ```bash
   helm package ./lifecycle-app
   ```
   This creates `lifecycle-app-0.1.0.tgz`.

5. Install from the packaged chart:
   ```bash
   helm install lifecycle-demo ./lifecycle-app-0.1.0.tgz \
     --namespace exercise-5 \
     --create-namespace \
     --wait
   ```

6. Run the tests:
   ```bash
   helm test lifecycle-demo --namespace exercise-5
   ```

7. Upgrade the release:
   ```bash
   helm upgrade lifecycle-demo ./lifecycle-app \
     --namespace exercise-5 \
     --set replicaCount=3 \
     --wait
   ```

8. View the history:
   ```bash
   helm history lifecycle-demo --namespace exercise-5
   ```

9. Roll back to revision 1:
   ```bash
   helm rollback lifecycle-demo 1 --namespace exercise-5
   ```

10. Verify the rollback:
    ```bash
    helm history lifecycle-demo --namespace exercise-5
    kubectl get pods --namespace exercise-5
    ```

11. Get the release manifest and values:
    ```bash
    helm get manifest lifecycle-demo --namespace exercise-5
    helm get values lifecycle-demo --namespace exercise-5 --all
    ```

12. Uninstall with history preserved:
    ```bash
    helm uninstall lifecycle-demo --namespace exercise-5 --keep-history
    ```

13. Verify the history is still available:
    ```bash
    helm list --namespace exercise-5 --all
    helm history lifecycle-demo --namespace exercise-5
    ```

14. Final cleanup:
    ```bash
    helm uninstall lifecycle-demo --namespace exercise-5 2>/dev/null
    kubectl delete namespace exercise-5
    rm -rf lifecycle-app/ lifecycle-app-0.1.0.tgz
    ```

---

## Further Reading

- [Official Helm Documentation](https://helm.sh/docs/)
- [Helm Chart Best Practices Guide](https://helm.sh/docs/chart_best_practices/)
- [Artifact Hub -- Discover Helm Charts](https://artifacthub.io/)
- [Go Template Documentation](https://pkg.go.dev/text/template)
- [Sprig Template Functions](https://masterminds.github.io/sprig/)
