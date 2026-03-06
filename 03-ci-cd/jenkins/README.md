# Jenkins - Continuous Integration and Continuous Delivery

## Table of Contents

- [1. Introduction to CI/CD](#1-introduction-to-cicd)
- [2. Introduction to Jenkins](#2-introduction-to-jenkins)
- [3. Installing Jenkins](#3-installing-jenkins)
- [4. Jenkins Dashboard and Navigation](#4-jenkins-dashboard-and-navigation)
- [5. Pushing Code to GitHub](#5-pushing-code-to-github)
- [6. Configuring Jenkins with GitHub](#6-configuring-jenkins-with-github)
- [7. Creating Jenkins Pipelines](#7-creating-jenkins-pipelines)
- [8. Writing and Pushing Jenkinsfile](#8-writing-and-pushing-jenkinsfile)
- [9. Configuring Jenkins-GitHub Triggers](#9-configuring-jenkins-github-triggers)
- [10. Running Pipelines](#10-running-pipelines)
- [11. Practice Exercises](#11-practice-exercises)

---

## 1. Introduction to CI/CD

### 1.1 What is Continuous Integration?

**Continuous Integration (CI)** is a software development practice where developers frequently merge their code changes into a shared repository -- often multiple times per day. Each merge triggers an automated build and test process, which verifies that the new code does not break the existing codebase.

The core principles of CI are:

- **Frequent commits**: Developers push small, incremental changes rather than large, infrequent updates.
- **Automated builds**: Every commit triggers an automated build process that compiles the code (if applicable) and produces artifacts.
- **Automated testing**: The build process runs a suite of automated tests (unit tests, integration tests, linting) to validate the changes.
- **Fast feedback**: Developers receive immediate feedback on whether their changes introduced any issues.
- **Single source of truth**: The main branch (or trunk) always reflects a working state of the software.

**Without CI**, developers work in isolation on feature branches for days or weeks. When they finally merge, conflicts pile up, bugs hide in large diffs, and the integration process becomes painful -- often called "integration hell."

**With CI**, integration happens continuously. Problems are caught early when they are small, cheap, and easy to fix.

### 1.2 What is Continuous Delivery vs Continuous Deployment?

These two terms are often confused, but they represent different levels of automation:

**Continuous Delivery (CD)** extends CI by automatically preparing code changes for release to production. After the CI pipeline passes (build + test), the code is automatically deployed to a staging or pre-production environment. However, the final deployment to production requires a **manual approval** step -- a human clicks a button to release.

**Continuous Deployment** takes it one step further. Every change that passes all stages of the pipeline is automatically deployed to production with **no manual intervention**. There is no human gate between a successful build and production.

```
Continuous Integration:
  Code Commit --> Build --> Test --> [DONE]

Continuous Delivery:
  Code Commit --> Build --> Test --> Deploy to Staging --> [MANUAL APPROVAL] --> Deploy to Production

Continuous Deployment:
  Code Commit --> Build --> Test --> Deploy to Staging --> Deploy to Production (automatic)
```

**Which should you use?**

- **Continuous Delivery** is appropriate for most organizations. It provides safety through the manual approval step while still automating the hard parts.
- **Continuous Deployment** is used by mature engineering organizations with comprehensive automated test suites and robust monitoring. Companies like Netflix, Amazon, and Etsy deploy hundreds of times per day using continuous deployment.

### 1.3 Why CI/CD Matters in DevOps

CI/CD is the backbone of DevOps. It bridges the gap between development and operations by:

1. **Reducing risk**: Small, frequent releases are easier to debug and roll back than massive quarterly releases.
2. **Accelerating delivery**: Automation eliminates manual, error-prone steps, allowing teams to ship features faster.
3. **Improving quality**: Automated testing catches bugs before they reach production.
4. **Enabling collaboration**: Everyone works on a shared codebase with clear, automated rules for integration.
5. **Building confidence**: Teams can deploy on a Friday afternoon without fear because the pipeline validates everything.
6. **Shortening feedback loops**: Developers learn about problems in minutes, not days.
7. **Standardizing processes**: Every change goes through the same pipeline, regardless of who wrote it.

### 1.4 Where Jenkins Fits in the CI/CD Landscape

The CI/CD tool landscape is broad:

| Tool | Type | Hosting | Key Strength |
|------|------|---------|-------------|
| **Jenkins** | Self-hosted | On-premise / Cloud VM | Flexibility, plugin ecosystem |
| GitHub Actions | Cloud-hosted | GitHub | Tight GitHub integration |
| GitLab CI/CD | Cloud / Self-hosted | GitLab | Built into GitLab platform |
| CircleCI | Cloud-hosted | Cloud | Speed, Docker-native |
| Travis CI | Cloud-hosted | Cloud | Open-source project friendly |
| Azure DevOps | Cloud / Self-hosted | Azure | Microsoft ecosystem |
| ArgoCD | Self-hosted | Kubernetes | GitOps, Kubernetes-native CD |

**Jenkins** is the most widely adopted CI/CD tool in the world. It is:

- **Open source** and free to use.
- **Self-hosted**, meaning you control the infrastructure, data, and configuration.
- **Highly extensible** with over 1,800 plugins covering virtually every tool and platform.
- **Battle-tested** with over 15 years of production use.
- **Cloud-agnostic** -- it runs on AWS, GCP, Azure, on-premise servers, or even a Raspberry Pi.

Jenkins is an excellent starting point for learning CI/CD because it teaches you the underlying concepts without abstracting them away. Once you understand Jenkins pipelines, transitioning to GitHub Actions, GitLab CI, or any other tool becomes straightforward.

---

## 2. Introduction to Jenkins

### 2.1 What is Jenkins?

Jenkins is an open-source automation server written in Java. It provides hundreds of plugins to support building, deploying, and automating any project.

**Brief History:**

- **2004**: Kohsuke Kawaguchi at Sun Microsystems created **Hudson**, a CI server, as a side project.
- **2011**: After Oracle acquired Sun, a dispute over the project's direction led the community to fork Hudson and rename it **Jenkins**.
- **2011-present**: Jenkins became the dominant CI/CD tool, governed by the Jenkins community and the Continuous Delivery Foundation (CDF).
- **2016**: The **Pipeline plugin** (and Jenkinsfile) was introduced, transforming Jenkins from a UI-configured tool into a code-driven automation platform.
- **2017**: **Blue Ocean** was released, providing a modern UI for Jenkins pipelines.
- **Today**: Jenkins has over 300,000 installations worldwide and remains the most popular CI/CD server.

**The Jenkins Ecosystem:**

Jenkins is not a single tool -- it is a platform. The core provides job scheduling, a web UI, and an API. Everything else comes from plugins:

- **Pipeline plugins** for defining CI/CD as code
- **SCM plugins** for Git, SVN, Mercurial integration
- **Cloud plugins** for AWS, GCP, Azure, Kubernetes integration
- **Notification plugins** for Slack, email, Microsoft Teams alerts
- **Artifact plugins** for Docker, Nexus, Artifactory integration
- **Security plugins** for LDAP, Active Directory, Role-Based Access Control

### 2.2 Jenkins Architecture

Understanding Jenkins architecture is essential for effective use:

```
                    +-------------------+
                    |   Jenkins Master  |
                    |   (Controller)    |
                    |                   |
                    |  - Scheduling     |
                    |  - UI / API       |
                    |  - Configuration  |
                    |  - Plugin mgmt    |
                    +--------+----------+
                             |
              +--------------+--------------+
              |              |              |
        +-----+----+  +-----+----+  +-----+----+
        |  Agent 1 |  |  Agent 2 |  |  Agent 3 |
        |  (Linux) |  | (Windows)|  | (Docker) |
        |          |  |          |  |          |
        | Executor |  | Executor |  | Executor |
        | Executor |  | Executor |  | Executor |
        +----------+  +----------+  +----------+
```

**Master (Controller):**
The Jenkins master is the central server. It is responsible for:
- Hosting the Jenkins web UI and REST API
- Managing configuration, jobs, and plugins
- Scheduling builds and dispatching them to agents
- Monitoring agents and collecting build results
- Storing build logs, artifacts, and metadata

**Agents (Nodes):**
Agents are machines (physical, virtual, or containers) that execute the actual build work. They:
- Connect to the master via SSH, JNLP, or WebSocket
- Run build steps as instructed by the master
- Can be labeled (e.g., `linux`, `docker`, `gpu`) so jobs can target specific agent types
- Can be permanent (always running) or cloud-provisioned (spun up on demand)

**Executors:**
An executor is a single thread on an agent that can run one build at a time. If an agent has 2 executors, it can run 2 builds concurrently. The master also has executors, but best practice is to run builds on agents, not the master.

**For learning purposes**, you will use a single Jenkins instance where the master also acts as the agent. This is perfectly fine for development and learning.

### 2.3 Key Concepts

**Job (Project):**
A job is a runnable task in Jenkins. It defines what to do (build steps), when to do it (triggers), and what to do afterward (post-build actions). There are several job types:
- **Freestyle Job**: A simple, UI-configured job. Good for basic tasks.
- **Pipeline Job**: A job defined by a Jenkinsfile (code). The modern, recommended approach.
- **Multibranch Pipeline**: Automatically discovers branches in a repo and creates pipeline jobs for each.
- **Folder**: An organizational container for grouping related jobs.

**Build:**
A build is a single execution of a job. Each build has:
- A unique build number (e.g., #1, #2, #3)
- A status: SUCCESS (blue/green), FAILURE (red), UNSTABLE (yellow), ABORTED (grey)
- Console output (the log of everything that happened)
- Optional artifacts (files produced by the build)

**Pipeline:**
A pipeline is a suite of automated steps that take code from version control to production. In Jenkins, pipelines are defined in a `Jenkinsfile` and consist of:
- **Stages**: Logical groupings (e.g., "Build", "Test", "Deploy")
- **Steps**: Individual commands within a stage (e.g., `sh 'npm install'`)

**Plugin:**
A plugin extends Jenkins functionality. Jenkins ships with a minimal core -- nearly everything is a plugin, including Git support, pipeline syntax, credentials management, and the UI.

### 2.4 Jenkins vs Other CI/CD Tools

| Feature | Jenkins | GitHub Actions | GitLab CI/CD |
|---------|---------|---------------|-------------|
| **Hosting** | Self-hosted | Cloud (GitHub) | Cloud or self-hosted |
| **Configuration** | Jenkinsfile (Groovy) | YAML workflow files | `.gitlab-ci.yml` (YAML) |
| **Cost** | Free (you pay for infrastructure) | Free tier + paid minutes | Free tier + paid minutes |
| **Plugins** | 1,800+ plugins | Marketplace actions | Built-in features |
| **Learning curve** | Steeper | Moderate | Moderate |
| **Flexibility** | Extremely high | High (within GitHub) | High (within GitLab) |
| **UI** | Dated (Blue Ocean improves it) | Modern | Modern |
| **SCM integration** | Any Git host | GitHub only | GitLab only |
| **Best for** | Complex, custom workflows | GitHub-native projects | GitLab-native projects |

**Why learn Jenkins first?**
Jenkins forces you to understand the mechanics of CI/CD -- how agents work, how builds are scheduled, how plugins interact, and how pipelines are structured. Other tools abstract these details. Starting with Jenkins gives you transferable knowledge that applies to any CI/CD platform.

---

## 3. Installing Jenkins

Jenkins requires **Java 17** or **Java 21** to run. The installation method you choose depends on your operating system and preference. For learning, **Docker is the recommended method** because it is clean, portable, and easy to tear down.

### 3.1 macOS (Using Homebrew)

```bash
# Install the LTS (Long Term Support) version of Jenkins
brew install jenkins-lts
```

**Start Jenkins:**

```bash
# Start Jenkins as a background service
brew services start jenkins-lts
```

**Stop Jenkins:**

```bash
# Stop the Jenkins service
brew services stop jenkins-lts
```

**Restart Jenkins:**

```bash
# Restart the Jenkins service
brew services restart jenkins-lts
```

**Access Jenkins:**

Open your browser and navigate to:

```
http://localhost:8080
```

**Notes:**
- Jenkins data is stored in `~/.jenkins/` by default on macOS.
- The initial admin password is located at `~/.jenkins/secrets/initialAdminPassword`.
- If you need Java, install it first: `brew install openjdk@17`

```bash
# Verify Java is installed
java -version

# If not, install Java 17
brew install openjdk@17

# Add Java to your PATH (add this to ~/.zshrc or ~/.bash_profile)
export PATH="/opt/homebrew/opt/openjdk@17/bin:$PATH"
```

### 3.2 Ubuntu/Debian Linux

**Step 1: Install Java**

```bash
# Update package index
sudo apt update

# Install Java 17 (required by Jenkins)
sudo apt install -y openjdk-17-jdk

# Verify Java installation
java -version
```

**Step 2: Add Jenkins Repository**

```bash
# Add the Jenkins GPG key
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

# Add the Jenkins apt repository
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
```

**Step 3: Install Jenkins**

```bash
# Update package index to include Jenkins repo
sudo apt update

# Install Jenkins
sudo apt install -y jenkins
```

**Step 4: Start and Enable Jenkins**

```bash
# Start Jenkins service
sudo systemctl start jenkins

# Enable Jenkins to start on boot
sudo systemctl enable jenkins

# Check Jenkins status
sudo systemctl status jenkins
```

**Step 5: Open Firewall Port**

```bash
# If using UFW (Uncomplicated Firewall)
sudo ufw allow 8080
sudo ufw status

# If using iptables
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
```

**Step 6: Get Initial Admin Password**

```bash
# Display the initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy this password -- you will need it when you first access the Jenkins UI at `http://your-server-ip:8080`.

### 3.3 CentOS/RHEL/Fedora

**Step 1: Install Java**

```bash
# CentOS/RHEL
sudo yum install -y java-17-openjdk java-17-openjdk-devel

# Fedora
sudo dnf install -y java-17-openjdk java-17-openjdk-devel

# Verify Java installation
java -version
```

**Step 2: Add Jenkins Repository**

```bash
# Add Jenkins repo
sudo wget -O /etc/yum.repos.d/jenkins.repo \
  https://pkg.jenkins.io/redhat-stable/jenkins.repo

# Import Jenkins GPG key
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
```

**Step 3: Install Jenkins**

```bash
# CentOS/RHEL
sudo yum install -y jenkins

# Fedora
sudo dnf install -y jenkins
```

**Step 4: Start and Enable Jenkins**

```bash
# Reload systemd daemon (important after installing new service)
sudo systemctl daemon-reload

# Start Jenkins
sudo systemctl start jenkins

# Enable Jenkins to start on boot
sudo systemctl enable jenkins

# Verify Jenkins is running
sudo systemctl status jenkins
```

**Step 5: Configure Firewall**

```bash
# If using firewalld
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

# Verify the port is open
sudo firewall-cmd --list-ports
```

**Step 6: Get Initial Admin Password**

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Access Jenkins at `http://your-server-ip:8080`.

### 3.4 Windows

**Step 1: Install Java**

1. Download Java 17 JDK from https://adoptium.net/ (Eclipse Temurin).
2. Run the installer and follow the prompts.
3. Verify by opening Command Prompt and typing: `java -version`

**Step 2: Download and Install Jenkins**

1. Go to https://www.jenkins.io/download/ and download the Windows installer (`.msi` file).
2. Run the installer.
3. Choose "Run service as local or domain user" (recommended for production) or "Run service as LocalSystem" (fine for learning).
4. Set the port (default is 8080).
5. Select the Java installation path.
6. Complete the installation.

**Step 3: Verify the Service**

1. Open **Services** (type `services.msc` in the Start menu).
2. Find **Jenkins** in the list.
3. Ensure the status is **Running** and the startup type is **Automatic**.

**Step 4: Get Initial Admin Password**

Open Command Prompt or PowerShell:

```powershell
type "C:\ProgramData\Jenkins\.jenkins\secrets\initialAdminPassword"
```

Or navigate to that file in Windows Explorer and open it with Notepad.

Access Jenkins at `http://localhost:8080`.

### 3.5 Docker (Recommended for Learning)

Docker is the cleanest way to run Jenkins for learning purposes. It keeps Jenkins isolated from your system, and you can destroy and recreate it at will.

**Prerequisites:** Docker must be installed on your system. If you followed the Docker section of this DevOps-101 guide, you are ready.

#### Basic Docker Run

```bash
# Pull the official Jenkins LTS image
docker pull jenkins/jenkins:lts

# Run Jenkins in a container
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

**Explanation of flags:**

| Flag | Purpose |
|------|---------|
| `-d` | Run in detached mode (background) |
| `--name jenkins` | Name the container "jenkins" |
| `-p 8080:8080` | Map host port 8080 to container port 8080 (Jenkins UI) |
| `-p 50000:50000` | Map port 50000 for Jenkins agent communication |
| `-v jenkins_home:/var/jenkins_home` | Persist Jenkins data in a Docker named volume |

**Get the initial admin password:**

```bash
# View the initial admin password from the container logs
docker logs jenkins

# Or read it directly from the container
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

**Managing the container:**

```bash
# Stop Jenkins
docker stop jenkins

# Start Jenkins
docker start jenkins

# Restart Jenkins
docker restart jenkins

# Remove Jenkins container (data persists in volume)
docker rm -f jenkins

# Remove Jenkins data volume (WARNING: deletes all Jenkins data)
docker volume rm jenkins_home
```

#### Docker Compose for Jenkins

For a more manageable setup, use Docker Compose. Create a file called `docker-compose.yml`:

```yaml
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
    environment:
      - JAVA_OPTS=-Xmx2048m -Xms512m

volumes:
  jenkins_home:
    driver: local
```

```bash
# Start Jenkins with Docker Compose
docker compose up -d

# View logs
docker compose logs -f jenkins

# Stop Jenkins
docker compose down

# Stop and remove volumes (WARNING: deletes all data)
docker compose down -v
```

#### Jenkins with Docker-in-Docker (DinD) vs Docker Socket Mounting

When your Jenkins pipelines need to build Docker images, you have two options:

**Option A: Docker Socket Mounting (Recommended for learning)**

This approach shares the host's Docker daemon with the Jenkins container. Jenkins runs Docker commands, but they execute on the host.

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(which docker):/usr/bin/docker \
  --user root \
  jenkins/jenkins:lts
```

**Pros:** Simple, fast, no overhead.
**Cons:** Jenkins can access all containers on the host (security concern in production).

**Option B: Docker-in-Docker (DinD)**

This approach runs a separate Docker daemon inside the Jenkins container. More isolated but more complex.

```yaml
# docker-compose.yml with DinD
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
    environment:
      - DOCKER_HOST=tcp://docker:2376
      - DOCKER_CERT_PATH=/certs/client
      - DOCKER_TLS_VERIFY=1
    depends_on:
      - docker

  docker:
    image: docker:dind
    container_name: jenkins-docker
    privileged: true
    restart: unless-stopped
    environment:
      - DOCKER_TLS_CERTDIR=/certs
    volumes:
      - jenkins_home:/var/jenkins_home
      - docker-certs:/certs
    ports:
      - "2376:2376"

volumes:
  jenkins_home:
    driver: local
  docker-certs:
    driver: local
```

```bash
# Start the stack
docker compose up -d

# Verify both containers are running
docker compose ps
```

**For learning purposes, Docker socket mounting (Option A) is simpler and sufficient.**

### 3.6 Post-Installation Setup

Regardless of how you installed Jenkins, the first-time setup process is the same.

#### Step 1: Unlock Jenkins

1. Open your browser and go to `http://localhost:8080` (or your server's IP).
2. You will see the **Unlock Jenkins** screen asking for the initial admin password.
3. Retrieve the password:

```bash
# macOS (Homebrew)
cat ~/.jenkins/secrets/initialAdminPassword

# Linux (apt/yum)
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Docker
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

4. Paste the password into the form and click **Continue**.

#### Step 2: Install Plugins

You will be presented with two options:

- **Install suggested plugins**: Installs the most commonly used plugins. **Choose this option** for your first setup.
- **Select plugins to install**: Lets you pick specific plugins.

Click **Install suggested plugins** and wait for the installation to complete. This typically takes 2-5 minutes.

#### Step 3: Create Admin User

Fill in the form:
- **Username**: `admin` (or your preferred username)
- **Password**: Choose a strong password
- **Full name**: Your full name
- **E-mail address**: Your email

Click **Save and Continue**.

#### Step 4: Configure Jenkins URL

Jenkins will ask for the **Jenkins URL**. This is the URL that Jenkins uses to refer to itself (in emails, webhook callbacks, etc.).

- For local development: `http://localhost:8080/`
- For a server: `http://your-server-ip:8080/`

Click **Save and Finish**, then **Start using Jenkins**.

#### Step 5: Install Essential Plugins

After the initial setup, install these additional plugins:

1. Go to **Manage Jenkins** (left sidebar) > **Plugins** > **Available plugins**.
2. Search for and install the following:

| Plugin | Purpose |
|--------|---------|
| **Git** | Git SCM integration (usually pre-installed) |
| **Pipeline** | Pipeline-as-code support (usually pre-installed) |
| **Docker Pipeline** | Build and use Docker containers in pipelines |
| **Docker Commons** | Shared Docker functionality for plugins |
| **GitHub Integration** | GitHub webhook and status integration |
| **GitHub Branch Source** | Multibranch pipeline support for GitHub |
| **Credentials** | Credential storage and management (usually pre-installed) |
| **Credentials Binding** | Inject credentials into build steps |
| **Blue Ocean** | Modern pipeline visualization UI |
| **Pipeline: Stage View** | Visualize pipeline stages |
| **Workspace Cleanup** | Clean workspace before/after builds |
| **Timestamper** | Add timestamps to console output |
| **SSH Agent** | SSH key management for builds |

3. Click **Install** and check **Restart Jenkins when installation is complete and no jobs are running**.

After Jenkins restarts, log back in with your admin credentials.

---

## 4. Jenkins Dashboard and Navigation

### 4.1 Overview of the Jenkins UI

After logging in, you will see the Jenkins dashboard:

```
+------------------------------------------------------------------+
|  Jenkins                                    [user] | [log out]   |
+------------------------------------------------------------------+
|                                                                   |
|  LEFT SIDEBAR:             MAIN AREA:                            |
|  +------------------+      +----------------------------------+  |
|  | New Item          |      | Welcome to Jenkins!              |  |
|  | People            |      |                                  |  |
|  | Build History     |      | Build Queue: (empty)             |  |
|  | Manage Jenkins    |      | Build Executor Status:           |  |
|  | My Views          |      |   Master: Idle  Idle             |  |
|  +------------------+      |                                  |  |
|                            | JOBS LIST:                        |  |
|                            | (No jobs yet)                     |  |
|                            +----------------------------------+  |
+------------------------------------------------------------------+
```

**Left Sidebar:**
- **New Item**: Create a new job/pipeline
- **People**: View users and their activity
- **Build History**: See recent builds across all jobs
- **Manage Jenkins**: System configuration, plugin management, security settings
- **My Views**: Custom dashboard views

**Main Area:**
- **Build Queue**: Jobs waiting to be executed
- **Build Executor Status**: Shows what each executor is doing
- **Job List**: All configured jobs with their status

### 4.2 Managing Jenkins Settings

Navigate to **Manage Jenkins** to access system configuration:

**System Configuration:**
- **System**: Global settings like Jenkins URL, email notification, environment variables
- **Tools**: Configure JDK, Git, Maven, Docker, Node.js paths
- **Plugins**: Install, update, and manage plugins
- **Nodes**: Manage build agents and executors

**Security:**
- **Security**: Authentication, authorization, CSRF protection
- **Credentials**: Manage stored secrets (passwords, tokens, SSH keys)
- **Users**: Manage Jenkins user accounts

### 4.3 Global Tool Configuration

Go to **Manage Jenkins** > **Tools** to configure tools that Jenkins pipelines will use.

**Git Configuration:**
- Name: `Default`
- Path to Git executable: `git` (or the full path like `/usr/bin/git`)
- Jenkins usually auto-detects Git if it is in the system PATH.

**JDK Configuration:**
- Click **Add JDK**
- Name: `JDK17`
- JAVA_HOME: `/usr/lib/jvm/java-17-openjdk-amd64` (Linux) or `/opt/homebrew/opt/openjdk@17` (macOS)

**Docker Configuration (if Docker Pipeline plugin is installed):**
- Name: `Docker`
- Installation root: `/usr/bin` (or wherever Docker is installed)

### 4.4 Credentials Management

Credentials are how Jenkins securely stores and uses secrets. Go to **Manage Jenkins** > **Credentials** > **System** > **Global credentials (unrestricted)** > **Add Credentials**.

#### Adding a GitHub Personal Access Token

1. **Kind**: Username with password
2. **Scope**: Global
3. **Username**: Your GitHub username
4. **Password**: Your GitHub Personal Access Token (PAT)
5. **ID**: `github-credentials` (you will reference this ID in pipelines)
6. **Description**: `GitHub Personal Access Token`
7. Click **Create**

#### Adding Docker Hub Credentials

1. **Kind**: Username with password
2. **Scope**: Global
3. **Username**: Your Docker Hub username
4. **Password**: Your Docker Hub password or access token
5. **ID**: `dockerhub-credentials`
6. **Description**: `Docker Hub Credentials`
7. Click **Create**

#### Adding an SSH Private Key

1. **Kind**: SSH Username with private key
2. **Scope**: Global
3. **Username**: `git`
4. **Private Key**: Enter directly > paste your private key
5. **ID**: `github-ssh-key`
6. **Description**: `GitHub SSH Key`
7. Click **Create**

#### Adding a Secret Text (API Token)

1. **Kind**: Secret text
2. **Scope**: Global
3. **Secret**: Paste the token value
4. **ID**: `my-api-token`
5. **Description**: `API Token for Service X`
6. Click **Create**

You will reference these credential IDs in your Jenkinsfile using `credentials('github-credentials')` or `withCredentials` blocks.

---

## 5. Pushing Code to GitHub

Before Jenkins can build your code, the code must live in a Git repository on GitHub. This section covers creating a repository, pushing your application code, and structuring the repository correctly.

### 5.1 Creating a GitHub Repository

1. Go to https://github.com and log in.
2. Click the **+** icon in the top-right corner and select **New repository**.
3. Fill in the details:
   - **Repository name**: `python-flask-app` (or any name for your project)
   - **Description**: `A Python Flask application with Docker and Jenkins CI`
   - **Visibility**: Public (or Private -- both work with Jenkins)
   - **Initialize**: Do NOT check "Add a README file" (we will push existing code)
4. Click **Create repository**.
5. Copy the repository URL (HTTPS or SSH).

### 5.2 Setting Up Authentication

#### Option A: Personal Access Token (HTTPS)

1. Go to https://github.com/settings/tokens (or GitHub > Settings > Developer settings > Personal access tokens > Tokens (classic)).
2. Click **Generate new token (classic)**.
3. Give it a descriptive name: `Jenkins CI Token`
4. Set expiration (e.g., 90 days).
5. Select scopes:
   - `repo` (full control of private repositories)
   - `admin:repo_hook` (manage webhooks)
   - `admin:org_hook` (if using organizations)
6. Click **Generate token**.
7. **Copy the token immediately** -- you will not be able to see it again.

When cloning or pushing, use the token as your password:

```bash
# Clone using PAT
git clone https://github.com/yourusername/python-flask-app.git
# When prompted for password, paste your Personal Access Token

# Or configure Git to use the token
git remote set-url origin https://yourusername:YOUR_TOKEN@github.com/yourusername/python-flask-app.git
```

#### Option B: SSH Keys (Recommended)

```bash
# Generate an SSH key pair (if you do not already have one)
ssh-keygen -t ed25519 -C "your.email@example.com"

# Start the SSH agent
eval "$(ssh-agent -s)"

# Add your private key to the SSH agent
ssh-add ~/.ssh/id_ed25519

# Copy the public key to clipboard
# macOS:
cat ~/.ssh/id_ed25519.pub | pbcopy

# Linux:
cat ~/.ssh/id_ed25519.pub | xclip -selection clipboard

# Windows (Git Bash):
cat ~/.ssh/id_ed25519.pub | clip
```

Add the public key to GitHub:
1. Go to https://github.com/settings/keys
2. Click **New SSH key**
3. Title: `My Laptop` (or any descriptive name)
4. Key type: Authentication Key
5. Paste the public key
6. Click **Add SSH key**

Test the connection:

```bash
ssh -T git@github.com
# Expected output: "Hi yourusername! You've successfully authenticated..."
```

### 5.3 Pushing the Python Application to GitHub

Let us assume you have a Python Flask application (from the Docker section of this guide). Here is the project structure we will push:

```
python-flask-app/
├── app.py
├── requirements.txt
├── Dockerfile
├── Jenkinsfile          (we will create this later)
├── tests/
│   └── test_app.py
├── .gitignore
└── README.md
```

**Sample application files:**

`app.py`:
```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({"message": "Hello from DevOps-101!", "status": "running"})

@app.route('/health')
def health():
    return jsonify({"status": "healthy"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

`requirements.txt`:
```
flask==3.0.0
pytest==7.4.3
```

`tests/test_app.py`:
```python
import pytest
from app import app

@pytest.fixture
def client():
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_home(client):
    response = client.get('/')
    assert response.status_code == 200
    data = response.get_json()
    assert data['status'] == 'running'

def test_health(client):
    response = client.get('/health')
    assert response.status_code == 200
    data = response.get_json()
    assert data['status'] == 'healthy'
```

`Dockerfile`:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

**Initialize and push:**

```bash
# Navigate to your project directory
cd python-flask-app

# Initialize a Git repository
git init -b main

# Create .gitignore (see next section)
# Add all files
git add .

# Create the first commit
git commit -m "Initial commit: Flask app with Docker support"

# Add the remote repository
git remote add origin git@github.com:yourusername/python-flask-app.git

# Push to GitHub
git push -u origin main
```

### 5.4 Repository Structure Best Practices

A well-structured repository makes CI/CD easier:

```
project-root/
├── .github/                  # GitHub-specific configuration
│   └── CODEOWNERS
├── src/                      # Application source code (or app code at root)
│   └── ...
├── tests/                    # Test files
│   ├── unit/
│   └── integration/
├── docker/                   # Docker-related files (if multiple)
│   └── Dockerfile.prod
├── Dockerfile                # Main Dockerfile (at root for simplicity)
├── Jenkinsfile               # Jenkins pipeline definition (MUST be at root)
├── docker-compose.yml        # Docker Compose for local development
├── requirements.txt          # Python dependencies (or package.json, go.mod, etc.)
├── .gitignore                # Files to exclude from Git
├── .dockerignore             # Files to exclude from Docker builds
└── README.md                 # Project documentation
```

Key points:
- **Jenkinsfile** must be at the repository root for Jenkins to auto-discover it.
- Keep configuration files at the root level where tools expect them.
- Separate source code, tests, and infrastructure configuration.

### 5.5 Creating a Proper .gitignore

`.gitignore`:
```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
*.egg-info/
dist/
build/
*.egg
.eggs/
venv/
env/
.venv/

# Testing
.pytest_cache/
.coverage
htmlcov/
.tox/

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Environment variables
.env
.env.local
.env.production

# Docker
docker-compose.override.yml

# Jenkins (local)
.jenkins/

# Secrets (NEVER commit these)
*.pem
*.key
secrets/
credentials/
```

---

## 6. Configuring Jenkins with GitHub

This section connects Jenkins to your GitHub repository so that Jenkins can pull code and receive webhook notifications.

### 6.1 Installing GitHub Plugins in Jenkins

If you followed the post-installation setup, these should already be installed. Verify by going to **Manage Jenkins** > **Plugins** > **Installed plugins** and searching for:

- **Git plugin** (provides Git SCM integration)
- **GitHub plugin** (provides GitHub webhook support)
- **GitHub Branch Source plugin** (provides multibranch pipeline support)

If any are missing, go to **Available plugins**, search for them, install, and restart Jenkins.

### 6.2 Adding GitHub Credentials to Jenkins

**Step 1:** Go to **Manage Jenkins** > **Credentials** > **System** > **Global credentials (unrestricted)**.

**Step 2:** Click **Add Credentials**.

**Step 3:** Fill in the form:
- **Kind**: Username with password
- **Scope**: Global (Jenkins, nodes, items, all child items, etc.)
- **Username**: Your GitHub username (e.g., `sooraj`)
- **Password**: Paste your GitHub Personal Access Token (NOT your GitHub password)
- **ID**: `github-pat` (this is the identifier you will use in pipelines)
- **Description**: `GitHub Personal Access Token for CI`

**Step 4:** Click **Create**.

### 6.3 Testing the Connection

**Step 1:** Go to **Manage Jenkins** > **System**.

**Step 2:** Scroll down to the **GitHub** section (if using the GitHub plugin).

**Step 3:** Click **Add GitHub Server** > **GitHub Server**.

**Step 4:** Configure:
- **Name**: `GitHub`
- **API URL**: `https://api.github.com` (default, leave as-is)
- **Credentials**: Select the credential you just created

**Step 5:** Click **Test connection**. You should see a message like:

```
Credentials verified for user yourusername, rate limit: 4998/5000
```

**Step 6:** Check **Manage hooks** (this allows Jenkins to automatically manage webhooks on your GitHub repos).

**Step 7:** Click **Save**.

### 6.4 Setting Up GitHub Webhooks

Webhooks allow GitHub to notify Jenkins whenever code is pushed. Instead of Jenkins periodically checking for changes (polling), GitHub proactively sends a notification.

#### Setting Up Webhooks on GitHub

1. Go to your GitHub repository (e.g., `https://github.com/yourusername/python-flask-app`).
2. Click **Settings** (tab at the top of the repo page).
3. Click **Webhooks** in the left sidebar.
4. Click **Add webhook**.
5. Fill in the form:

| Field | Value |
|-------|-------|
| **Payload URL** | `http://your-jenkins-url:8080/github-webhook/` |
| **Content type** | `application/json` |
| **Secret** | (leave blank for now, or add a secret and configure it in Jenkins) |
| **Which events?** | Select **Just the push event** (or choose individual events) |
| **Active** | Check this box |

6. Click **Add webhook**.

GitHub will send a test ping. Go back to the webhook settings and you should see a green checkmark next to the recent delivery.

**Important:** The Payload URL must end with `/github-webhook/` (with the trailing slash). This is the endpoint that the Jenkins GitHub plugin listens on.

#### Recommended Events to Trigger

For most CI setups, select **Let me select individual events** and check:

- **Pushes** -- triggers on every push to any branch
- **Pull requests** -- triggers when PRs are opened, updated, or merged

### 6.5 For Local Development: Using ngrok

If Jenkins is running on `localhost`, GitHub cannot reach it because `localhost` is not accessible from the internet. **ngrok** creates a secure tunnel from a public URL to your local machine.

**Step 1: Install ngrok**

```bash
# macOS (Homebrew)
brew install ngrok

# Linux (snap)
sudo snap install ngrok

# Or download from https://ngrok.com/download
```

**Step 2: Create a Free Account**

1. Go to https://ngrok.com/ and sign up for a free account.
2. Get your auth token from the dashboard.

```bash
# Authenticate ngrok
ngrok config add-authtoken YOUR_AUTH_TOKEN
```

**Step 3: Start the Tunnel**

```bash
# Expose your local Jenkins on port 8080
ngrok http 8080
```

ngrok will display output like:

```
Forwarding   https://abc123.ngrok-free.app -> http://localhost:8080
```

**Step 4: Use the ngrok URL as the Webhook Payload URL**

In your GitHub webhook settings, set the Payload URL to:

```
https://abc123.ngrok-free.app/github-webhook/
```

**Important notes about ngrok:**
- The free tier gives you a random URL that changes every time you restart ngrok.
- You will need to update the GitHub webhook URL each time.
- ngrok is for development only -- do not use it in production.
- The tunnel is active only while the `ngrok` process is running.

---

## 7. Creating Jenkins Pipelines

### 7.1 Freestyle Jobs (Basic)

Freestyle jobs are the simplest type of Jenkins job. They are configured entirely through the UI.

#### Creating a Freestyle Job

**Step 1:** From the Jenkins dashboard, click **New Item** (left sidebar).

**Step 2:** Enter a name: `python-flask-app-freestyle`

**Step 3:** Select **Freestyle project** and click **OK**.

**Step 4: Configure Source Code Management**

In the job configuration page:
1. Scroll to **Source Code Management**.
2. Select **Git**.
3. Repository URL: `https://github.com/yourusername/python-flask-app.git`
4. Credentials: Select the GitHub credentials you added earlier (`github-pat`).
5. Branch Specifier: `*/main` (or `*/develop`, etc.)

**Step 5: Configure Build Triggers**

Scroll to **Build Triggers** and select one or more:

- **GitHub hook trigger for GITScm polling**: Triggers the build when GitHub sends a webhook. This is the recommended option.
- **Poll SCM**: Jenkins periodically checks for changes. Enter a cron schedule, e.g., `H/5 * * * *` (every 5 minutes). Less efficient than webhooks but works without exposing Jenkins to the internet.

**Step 6: Configure Build Steps**

Scroll to **Build Steps** and click **Add build step** > **Execute shell**.

Enter the build commands:

```bash
#!/bin/bash
echo "=== Starting Build ==="

# Install dependencies
pip install -r requirements.txt

# Run tests
python -m pytest tests/ -v

# Build Docker image
docker build -t python-flask-app:${BUILD_NUMBER} .

echo "=== Build Complete ==="
```

**Step 7: Configure Post-Build Actions**

Scroll to **Post-build actions** and add:
- **Publish JUnit test result report**: Test report XMLs: `**/test-results.xml`
- **Delete workspace when build is done**: Keeps things clean
- **E-mail Notification**: Enter email addresses to notify on failure

**Step 8:** Click **Save**.

**Step 9:** Click **Build Now** (left sidebar) to trigger the first build.

#### Limitations of Freestyle Jobs

While freestyle jobs work, they have significant limitations:
- Configuration is stored in Jenkins, not in your repository (not version-controlled).
- Difficult to replicate across Jenkins instances.
- No support for complex workflows (parallel stages, conditional logic).
- Cannot be reviewed in pull requests.

**This is why Pipeline jobs are the recommended approach.**

### 7.2 Pipeline Jobs (Recommended)

Pipeline jobs use a `Jenkinsfile` to define the entire build process as code. This file lives in your repository alongside your application code.

#### Creating a Pipeline Job

**Step 1:** From the Jenkins dashboard, click **New Item**.

**Step 2:** Enter a name: `python-flask-app-pipeline`

**Step 3:** Select **Pipeline** and click **OK**.

**Step 4: Configure the Pipeline**

Scroll to the **Pipeline** section. You have two options:

**Option A: Pipeline script (inline)**

Select **Pipeline script** and enter your pipeline directly in the text area. Good for testing, but the pipeline is stored in Jenkins, not in Git.

```groovy
pipeline {
    agent any

    stages {
        stage('Hello') {
            steps {
                echo 'Hello from Jenkins Pipeline!'
            }
        }
    }
}
```

**Option B: Pipeline script from SCM (Recommended)**

Select **Pipeline script from SCM** to load the Jenkinsfile from your Git repository:

1. **SCM**: Git
2. **Repository URL**: `https://github.com/yourusername/python-flask-app.git`
3. **Credentials**: Select your GitHub credentials
4. **Branch Specifier**: `*/main`
5. **Script Path**: `Jenkinsfile` (this is the default and assumes the Jenkinsfile is at the root of the repo)

**Step 5:** Click **Save**.

#### Declarative vs Scripted Pipeline Syntax

Jenkins supports two pipeline syntaxes:

**Declarative Pipeline (Recommended):**

Declarative syntax is structured, opinionated, and easier to read. It uses a predefined structure with `pipeline`, `agent`, `stages`, `stage`, and `steps` blocks.

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'echo Building...'
            }
        }
        stage('Test') {
            steps {
                sh 'echo Testing...'
            }
        }
        stage('Deploy') {
            steps {
                sh 'echo Deploying...'
            }
        }
    }
}
```

**Scripted Pipeline:**

Scripted syntax uses Groovy programming language and provides more flexibility but is harder to read and maintain.

```groovy
node {
    stage('Build') {
        sh 'echo Building...'
    }
    stage('Test') {
        sh 'echo Testing...'
    }
    stage('Deploy') {
        sh 'echo Deploying...'
    }
}
```

**Use Declarative Pipeline** unless you have a specific need for scripted pipeline's flexibility. Declarative is the standard for modern Jenkins usage.

#### Pipeline Stages and Steps

**Stages** are logical divisions of your pipeline. Common stages include:

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                // Pull code from Git (automatic in Pipeline from SCM)
                checkout scm
            }
        }

        stage('Build') {
            steps {
                // Compile, package, or build Docker image
                sh 'docker build -t myapp:latest .'
            }
        }

        stage('Test') {
            steps {
                // Run unit tests, integration tests, linting
                sh 'python -m pytest tests/ -v'
            }
        }

        stage('Push') {
            steps {
                // Push Docker image to registry
                sh 'docker push myapp:latest'
            }
        }

        stage('Deploy') {
            steps {
                // Deploy to staging or production
                sh 'echo Deploying to production...'
            }
        }
    }
}
```

**Steps** are the individual actions within a stage. Common steps include:

| Step | Purpose | Example |
|------|---------|---------|
| `sh` | Run a shell command | `sh 'npm install'` |
| `echo` | Print a message | `echo 'Build started'` |
| `checkout` | Check out source code | `checkout scm` |
| `git` | Clone a Git repo | `git url: 'https://...'` |
| `dir` | Change directory | `dir('subdir') { sh 'ls' }` |
| `withCredentials` | Inject secrets | See section 8 |
| `archiveArtifacts` | Save build artifacts | `archiveArtifacts 'dist/**'` |
| `junit` | Publish test results | `junit 'test-results.xml'` |
| `input` | Wait for user input | `input 'Deploy to prod?'` |
| `timeout` | Set a time limit | `timeout(time: 10, unit: 'MINUTES')` |
| `retry` | Retry on failure | `retry(3) { sh 'flaky-command' }` |

---

## 8. Writing and Pushing Jenkinsfile

The `Jenkinsfile` is the heart of Jenkins Pipeline-as-Code. It defines your entire CI/CD pipeline in a single file that lives in your repository.

### 8.1 Jenkinsfile Syntax Deep Dive

#### Basic Structure

```groovy
pipeline {
    agent any                    // WHERE to run (any available agent)

    environment {                // ENVIRONMENT VARIABLES
        APP_NAME = 'python-flask-app'
        DOCKER_IMAGE = "yourusername/${APP_NAME}"
    }

    options {                    // PIPELINE OPTIONS
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    stages {                     // WHAT to do
        stage('Stage Name') {
            steps {
                // commands here
            }
        }
    }

    post {                       // WHAT to do after (always, on success, on failure)
        always {
            echo 'Pipeline finished.'
        }
    }
}
```

#### Agent Directive

The `agent` directive tells Jenkins where to run the pipeline or stage.

```groovy
// Run on any available agent
agent any

// Run on a specific labeled agent
agent { label 'linux' }

// Run inside a Docker container
agent {
    docker {
        image 'python:3.11-slim'
        args '-v /tmp:/tmp'
    }
}

// Run inside a Docker container built from a Dockerfile in the repo
agent {
    dockerfile {
        filename 'Dockerfile'
        dir 'docker'
        additionalBuildArgs '--build-arg HTTP_PROXY=http://proxy:8080'
    }
}

// Do not allocate an agent for the top-level pipeline
// (each stage specifies its own agent)
agent none
```

#### Environment Variables

```groovy
pipeline {
    agent any

    environment {
        // Static variables
        APP_NAME = 'my-app'
        APP_VERSION = '1.0.0'

        // Using Jenkins built-in variables
        BUILD_TAG = "${env.BUILD_TAG}"

        // Using credentials (automatically masked in logs)
        DOCKER_CREDENTIALS = credentials('dockerhub-credentials')
        // This creates three variables:
        //   DOCKER_CREDENTIALS       = username:password
        //   DOCKER_CREDENTIALS_USR   = username
        //   DOCKER_CREDENTIALS_PSW   = password

        // Secret text credential
        API_TOKEN = credentials('my-api-token')
    }

    stages {
        stage('Example') {
            environment {
                // Stage-level environment variable
                STAGE_VAR = 'only available in this stage'
            }
            steps {
                sh 'echo "Building ${APP_NAME} version ${APP_VERSION}"'
                sh 'echo "Build number: ${BUILD_NUMBER}"'
            }
        }
    }
}
```

**Jenkins Built-in Environment Variables:**

| Variable | Description | Example Value |
|----------|-------------|---------------|
| `BUILD_NUMBER` | Current build number | `42` |
| `BUILD_ID` | Same as BUILD_NUMBER | `42` |
| `BUILD_URL` | URL to the build | `http://jenkins:8080/job/my-app/42/` |
| `JOB_NAME` | Name of the job | `my-app` |
| `WORKSPACE` | Absolute path to the workspace | `/var/jenkins_home/workspace/my-app` |
| `JENKINS_URL` | URL of the Jenkins server | `http://jenkins:8080/` |
| `GIT_COMMIT` | Git commit SHA | `abc123def456...` |
| `GIT_BRANCH` | Git branch name | `origin/main` |
| `BRANCH_NAME` | Branch name (multibranch only) | `main` |

#### Using withCredentials

For more fine-grained credential injection, use `withCredentials`:

```groovy
stage('Push to Docker Hub') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'dockerhub-credentials',
                usernameVariable: 'DOCKER_USER',
                passwordVariable: 'DOCKER_PASS'
            )
        ]) {
            sh '''
                echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                docker logout
            '''
        }
    }
}
```

**Other withCredentials types:**

```groovy
// SSH key
withCredentials([sshUserPrivateKey(
    credentialsId: 'my-ssh-key',
    keyFileVariable: 'SSH_KEY',
    usernameVariable: 'SSH_USER'
)]) {
    sh 'ssh -i ${SSH_KEY} ${SSH_USER}@server.com "deploy.sh"'
}

// Secret text
withCredentials([string(
    credentialsId: 'my-api-token',
    variable: 'TOKEN'
)]) {
    sh 'curl -H "Authorization: Bearer ${TOKEN}" https://api.example.com'
}

// Secret file
withCredentials([file(
    credentialsId: 'kubeconfig',
    variable: 'KUBECONFIG'
)]) {
    sh 'kubectl get pods --kubeconfig=${KUBECONFIG}'
}
```

#### Post Section

The `post` section defines actions that run after stages complete, based on the build result:

```groovy
post {
    always {
        // Always runs, regardless of build result
        echo 'Pipeline completed.'
        cleanWs()  // Clean workspace
    }

    success {
        // Runs only if the pipeline succeeds
        echo 'Build succeeded!'
        // Send Slack notification, update deployment status, etc.
    }

    failure {
        // Runs only if the pipeline fails
        echo 'Build failed!'
        // Send alert, create incident ticket, etc.
    }

    unstable {
        // Runs if the build is marked unstable (e.g., test failures)
        echo 'Build is unstable.'
    }

    changed {
        // Runs if the current build status is different from the previous build
        echo 'Build status changed.'
    }

    cleanup {
        // Always runs, even if other post conditions fail
        // Use for critical cleanup (remove temp files, logout from registries)
        sh 'docker logout || true'
    }
}
```

#### Parameterized Builds

Parameters let users provide input when triggering a build:

```groovy
pipeline {
    agent any

    parameters {
        string(
            name: 'DEPLOY_ENV',
            defaultValue: 'staging',
            description: 'Target environment (staging or production)'
        )
        booleanParam(
            name: 'RUN_TESTS',
            defaultValue: true,
            description: 'Whether to run tests'
        )
        choice(
            name: 'DOCKER_TAG',
            choices: ['latest', 'stable', 'beta'],
            description: 'Docker image tag to use'
        )
    }

    stages {
        stage('Test') {
            when {
                expression { params.RUN_TESTS == true }
            }
            steps {
                sh 'python -m pytest tests/ -v'
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploying to ${params.DEPLOY_ENV} with tag ${params.DOCKER_TAG}"
            }
        }
    }
}
```

When you click **Build with Parameters**, Jenkins shows a form where you can fill in these values before the build starts.

### 8.2 Full Jenkinsfile for the Python/Docker Project

Here is a complete, production-style Jenkinsfile for the Python Flask application:

```groovy
pipeline {
    agent any

    environment {
        // Application settings
        APP_NAME = 'python-flask-app'
        DOCKER_IMAGE = "yourusername/${APP_NAME}"

        // Credentials (configured in Jenkins Credentials Manager)
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
    }

    options {
        // Abort the build if it takes longer than 30 minutes
        timeout(time: 30, unit: 'MINUTES')

        // Do not allow concurrent builds of the same job
        disableConcurrentBuilds()

        // Keep only the last 10 builds
        buildDiscarder(logRotator(numToKeepStr: '10'))

        // Add timestamps to console output
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checking out source code..."
                checkout scm
                sh 'ls -la'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image: ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                sh """
                    docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                    docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Run Tests') {
            steps {
                echo "Running tests inside Docker container..."
                sh """
                    docker run --rm ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                        python -m pytest tests/ -v --tb=short
                """
            }
        }

        stage('Push to Docker Hub') {
            when {
                branch 'main'
            }
            steps {
                echo "Pushing image to Docker Hub..."
                sh """
                    echo "${DOCKERHUB_CREDENTIALS_PSW}" | docker login -u "${DOCKERHUB_CREDENTIALS_USR}" --password-stdin
                    docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    docker push ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Cleanup') {
            steps {
                echo "Cleaning up local Docker images..."
                sh """
                    docker rmi ${DOCKER_IMAGE}:${BUILD_NUMBER} || true
                    docker rmi ${DOCKER_IMAGE}:latest || true
                """
            }
        }
    }

    post {
        always {
            echo "Build #${BUILD_NUMBER} completed with status: ${currentBuild.currentResult}"
            // Clean up the workspace
            cleanWs()
        }

        success {
            echo "Build SUCCEEDED. Docker image ${DOCKER_IMAGE}:${BUILD_NUMBER} is ready."
        }

        failure {
            echo "Build FAILED. Check the console output for errors."
            // In a real project, you would send a notification here:
            // slackSend channel: '#ci-cd', message: "Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            // mail to: 'team@example.com', subject: "FAILED: ${env.JOB_NAME}", body: "Check ${env.BUILD_URL}"
        }

        cleanup {
            // Ensure we always logout from Docker Hub
            sh 'docker logout || true'
        }
    }
}
```

### 8.3 Advanced Jenkinsfile Patterns

#### Parallel Stages

Run multiple stages simultaneously to save time:

```groovy
stage('Tests') {
    parallel {
        stage('Unit Tests') {
            steps {
                sh 'python -m pytest tests/unit/ -v'
            }
        }
        stage('Integration Tests') {
            steps {
                sh 'python -m pytest tests/integration/ -v'
            }
        }
        stage('Lint') {
            steps {
                sh 'python -m flake8 app.py --max-line-length=120'
            }
        }
    }
}
```

#### Conditional Stages with `when`

```groovy
stage('Deploy to Staging') {
    when {
        branch 'develop'
    }
    steps {
        echo 'Deploying to staging...'
    }
}

stage('Deploy to Production') {
    when {
        branch 'main'
        beforeInput true
    }
    input {
        message 'Deploy to production?'
        ok 'Yes, deploy!'
    }
    steps {
        echo 'Deploying to production...'
    }
}

stage('Run Only on Tags') {
    when {
        tag pattern: "v\\d+\\.\\d+\\.\\d+", comparator: "REGEXP"
    }
    steps {
        echo "Building release ${TAG_NAME}"
    }
}
```

#### Matrix Builds

Test across multiple configurations:

```groovy
stage('Test Matrix') {
    matrix {
        axes {
            axis {
                name 'PYTHON_VERSION'
                values '3.9', '3.10', '3.11'
            }
            axis {
                name 'OS'
                values 'ubuntu', 'alpine'
            }
        }
        stages {
            stage('Test') {
                agent {
                    docker { image "python:${PYTHON_VERSION}-${OS}" }
                }
                steps {
                    sh 'python -m pytest tests/ -v'
                }
            }
        }
    }
}
```

### 8.4 Pushing Jenkinsfile to the GitHub Repository

```bash
# Navigate to your project directory
cd python-flask-app

# Create the Jenkinsfile (copy the full Jenkinsfile from section 8.2 above)
# Save it as 'Jenkinsfile' at the root of the repository

# Stage the Jenkinsfile
git add Jenkinsfile

# Commit
git commit -m "Add Jenkinsfile for CI pipeline"

# Push to GitHub
git push origin main
```

Once pushed, Jenkins will detect the Jenkinsfile (if configured with "Pipeline script from SCM") and use it to define the pipeline.

---

## 9. Configuring Jenkins-GitHub Triggers

### 9.1 Poll SCM (Cron Syntax)

Poll SCM makes Jenkins periodically check the repository for changes. If changes are found, a build is triggered.

In the job configuration:
1. Scroll to **Build Triggers**.
2. Check **Poll SCM**.
3. Enter a cron schedule in the **Schedule** field.

**Cron syntax:**

```
MINUTE HOUR DAY_OF_MONTH MONTH DAY_OF_WEEK
```

**Examples:**

```
# Every 5 minutes
H/5 * * * *

# Every 15 minutes
H/15 * * * *

# Every hour
H * * * *

# Every day at midnight
H 0 * * *

# Every weekday at 8 AM
H 8 * * 1-5

# Every 2 hours
H H/2 * * *
```

The `H` symbol is Jenkins-specific. It means "hash" -- Jenkins picks a consistent minute within the range based on the job name, which distributes load across the server. `H/15` means "every 15 minutes, but the exact minute is chosen by Jenkins."

**Pros:** Works without exposing Jenkins to the internet. Simple setup.
**Cons:** Delayed feedback (up to 5 minutes if polling every 5 minutes). Unnecessary load on both Jenkins and GitHub.

### 9.2 GitHub Webhooks (Push Events)

Webhooks are the preferred trigger method. GitHub sends an HTTP POST to Jenkins immediately when code is pushed.

**Setup (if not done in Section 6):**

1. In the Jenkins job configuration, under **Build Triggers**, check **GitHub hook trigger for GITScm polling**.
2. In your GitHub repository, go to **Settings** > **Webhooks** > **Add webhook**.
3. Set the Payload URL to `http://your-jenkins-url:8080/github-webhook/`.
4. Content type: `application/json`.
5. Select **Just the push event**.
6. Click **Add webhook**.

**Verifying webhooks are working:**

After pushing a commit, go to your GitHub webhook settings and click on the webhook. Under **Recent Deliveries**, you should see:
- A green checkmark for successful delivery.
- The request payload (what GitHub sent).
- The response (what Jenkins replied).

If you see a red X, check:
- Is the Jenkins URL correct and reachable from the internet?
- Is the GitHub plugin installed in Jenkins?
- Does the payload URL end with `/github-webhook/` (trailing slash)?

### 9.3 GitHub Pull Request Triggers

To trigger builds when pull requests are opened, updated, or merged, you need the **GitHub Pull Request Builder** plugin or use **Multibranch Pipelines** (recommended).

With a Multibranch Pipeline (covered in the next section), pull request builds happen automatically. Jenkins discovers PRs, creates temporary pipeline jobs for them, and runs the Jenkinsfile against the PR branch.

### 9.4 Multibranch Pipeline Setup

A Multibranch Pipeline automatically discovers all branches (and optionally pull requests) in a repository and creates pipeline jobs for each one. This is the most powerful and recommended setup for real projects.

**Step 1:** From the Jenkins dashboard, click **New Item**.

**Step 2:** Enter a name: `python-flask-app-multibranch`

**Step 3:** Select **Multibranch Pipeline** and click **OK**.

**Step 4: Configure Branch Sources**

1. Under **Branch Sources**, click **Add source** > **GitHub**.
2. **Credentials**: Select your GitHub credentials.
3. **Repository HTTPS URL**: `https://github.com/yourusername/python-flask-app.git`
4. Click **Validate** to confirm Jenkins can access the repo.

**Step 5: Configure Behaviors**

Under **Behaviors**, you can configure:
- **Discover branches**: All branches, or only branches with PRs, or only named branches.
- **Discover pull requests from origin**: Builds PRs from the same repo.
- **Discover pull requests from forks**: Builds PRs from forks (be cautious with this -- malicious code could run in your Jenkins).

**Step 6: Configure Build Configuration**

- **Mode**: by Jenkinsfile
- **Script Path**: `Jenkinsfile`

**Step 7: Configure Scan Triggers**

Under **Scan Multibranch Pipeline Triggers**, set how often Jenkins scans for new branches:
- Check **Periodically if not otherwise run**
- Interval: `1 minute` (for development) or `1 hour` (for production)

**Step 8:** Click **Save**.

Jenkins will immediately scan the repository, discover all branches with a `Jenkinsfile`, and create pipeline jobs for each.

### 9.5 Branch Filtering and Discovery

You can control which branches Jenkins discovers and builds:

**In Multibranch Pipeline configuration, under Behaviors:**

**Filter by name (with regular expression):**
- Regular expression: `main|develop|feature/.*`
- This discovers only `main`, `develop`, and any branch starting with `feature/`.

**Filter by name (with wildcards):**
- Include: `main develop release/*`
- Exclude: `feature/experimental*`

**In Jenkinsfile, using `when` directive:**

```groovy
stage('Deploy to Production') {
    when {
        branch 'main'
    }
    steps {
        echo 'Deploying...'
    }
}

stage('Deploy to Staging') {
    when {
        branch 'develop'
    }
    steps {
        echo 'Deploying to staging...'
    }
}

stage('Run Only for Feature Branches') {
    when {
        branch pattern: "feature/.*", comparator: "REGEXP"
    }
    steps {
        echo 'Running feature branch tests...'
    }
}
```

---

## 10. Running Pipelines

### 10.1 Triggering Builds Manually

**From the job page:**
1. Navigate to the job (click on it from the dashboard).
2. Click **Build Now** (left sidebar) for non-parameterized builds.
3. Click **Build with Parameters** for parameterized builds, fill in the form, and click **Build**.

**From the dashboard:**
- Click the "play" button (right-pointing triangle icon) next to any job.

**From the command line (using Jenkins CLI or API):**

```bash
# Trigger a build using curl and Jenkins API
curl -X POST http://localhost:8080/job/python-flask-app-pipeline/build \
  --user admin:YOUR_API_TOKEN

# Trigger with parameters
curl -X POST http://localhost:8080/job/python-flask-app-pipeline/buildWithParameters \
  --user admin:YOUR_API_TOKEN \
  --data "DEPLOY_ENV=staging&RUN_TESTS=true"
```

### 10.2 Viewing Build Console Output

The console output is where you will spend most of your debugging time.

1. Navigate to the job.
2. Click on a specific build number (e.g., `#5`) in the build history.
3. Click **Console Output** (left sidebar).

The console output shows:
- Every command that was executed
- The output of each command (stdout and stderr)
- Timestamps (if the Timestamper plugin is installed)
- Credential values are automatically masked (shown as `****`)

**Tip:** For long console outputs, use **Full Log** instead of the truncated view. You can also download the log as a text file.

### 10.3 Blue Ocean Interface

Blue Ocean is a modern, visual interface for Jenkins pipelines.

**Accessing Blue Ocean:**
1. Click **Open Blue Ocean** in the left sidebar of the Jenkins dashboard.
2. Or navigate directly to `http://localhost:8080/blue`.

**Blue Ocean features:**
- Visual pipeline editor -- create pipelines without writing code
- Visual pipeline execution -- see each stage as a box with pass/fail status
- Branch and PR awareness -- see all branches and their build status
- Clear error highlighting -- failed stages are immediately visible
- Pipeline activity view -- a timeline of all builds

**Blue Ocean pipeline view:**

```
[Checkout] --> [Build] --> [Test] --> [Push] --> [Deploy]
    OK           OK         OK        OK        FAILED
```

Each stage shows:
- Green = passed
- Red = failed
- Blue = running
- Grey = not yet run or skipped

Click on any stage to see its console output. Click on a failed stage to immediately see the error.

### 10.4 Build Artifacts

Artifacts are files produced by a build that you want to preserve (e.g., compiled binaries, test reports, Docker images).

**Archiving artifacts in Jenkinsfile:**

```groovy
stage('Build') {
    steps {
        sh 'python setup.py sdist bdist_wheel'
    }
    post {
        success {
            archiveArtifacts artifacts: 'dist/**', fingerprint: true
        }
    }
}
```

**Viewing artifacts:**
1. Go to the build page (click on build number).
2. Artifacts are listed on the build page under **Build Artifacts**.
3. Click on any artifact to download it.

### 10.5 Build History and Trends

**Build History:**
- The left sidebar of every job shows a numbered list of builds with colored circles indicating status.
- Blue/green circle = success, red circle = failure, yellow circle = unstable, grey circle = aborted.
- Click any build to see its details.

**Build Trends:**
- Install the **Build Monitor View** plugin for a dashboard showing all job statuses.
- The **Pipeline Stage View** plugin shows a table of recent builds with stage durations.
- Test result trends are shown as graphs when JUnit test results are published.

### 10.6 Troubleshooting Failed Builds

Here are the most common errors and their solutions:

#### Error: "Permission denied"

```
+ docker build -t myapp:latest .
Got permission denied while trying to connect to the Docker daemon socket
```

**Solution:** The Jenkins user does not have permission to run Docker.

```bash
# Add the jenkins user to the docker group
sudo usermod -aG docker jenkins

# Restart Jenkins
sudo systemctl restart jenkins
```

If running Jenkins in Docker, mount the Docker socket and run as root (see section 3.5).

#### Error: "Command not found"

```
+ python -m pytest
/var/jenkins_home/workspace/my-job@tmp/durable-abc123/script.sh: line 1: python: not found
```

**Solution:** The required tool is not installed on the agent. Options:
- Install it on the agent.
- Use a Docker agent that has the tool pre-installed.
- Use the `tools` directive in your Jenkinsfile.

```groovy
pipeline {
    agent {
        docker { image 'python:3.11-slim' }
    }
    // ...
}
```

#### Error: "Authentication failed"

```
fatal: Authentication failed for 'https://github.com/...'
```

**Solution:** GitHub credentials are invalid or expired.
- Go to **Manage Jenkins** > **Credentials**.
- Update the password/token.
- Verify the credential ID matches what is used in the pipeline.

#### Error: "Jenkinsfile not found"

```
ERROR: Could not find Jenkinsfile at the root of the repository
```

**Solution:**
- Ensure the file is named exactly `Jenkinsfile` (capital J, no extension).
- Ensure it is in the root of the repository.
- Ensure it is committed and pushed to the branch Jenkins is building.
- Check the **Script Path** in the pipeline configuration.

#### Error: "No space left on device"

```
Error response from daemon: no space left on device
```

**Solution:** Jenkins builds accumulate Docker images and workspace data over time.

```bash
# Clean up Docker
docker system prune -af

# Clean old builds in Jenkins
# Go to Manage Jenkins > System > set workspace cleanup policies
# Or add cleanWs() to your Jenkinsfile post section
```

#### Error: "Script approval required"

```
Scripts not permitted to use method ... Administrators can decide whether to approve or reject this signature.
```

**Solution:** Jenkins sandboxes pipeline scripts. Some Groovy methods require approval.
- Go to **Manage Jenkins** > **In-process Script Approval**.
- Review and approve the pending signatures.

#### General Debugging Tips

1. **Always check the console output first.** The error is almost always visible there.
2. **Add `sh 'env'`** to your pipeline to print all environment variables -- useful for debugging credential and path issues.
3. **Add `sh 'ls -la'`** to verify the workspace contents at each stage.
4. **Use `catchError`** to continue the pipeline even if a stage fails:

```groovy
stage('Optional Stage') {
    steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
            sh 'some-command-that-might-fail'
        }
    }
}
```

5. **Use `script` blocks** for complex Groovy logic within declarative pipelines:

```groovy
stage('Debug') {
    steps {
        script {
            def files = sh(script: 'ls -la', returnStdout: true).trim()
            echo "Workspace contents:\n${files}"
        }
    }
}
```

---

## 11. Practice Exercises

### Exercise 1: Install Jenkins and Run Your First Build

**Objective:** Install Jenkins using Docker and run a basic pipeline.

**Steps:**

1. Install Jenkins using Docker:

```bash
docker run -d \
  --name jenkins-exercise \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_exercise_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

2. Get the initial admin password:

```bash
docker exec jenkins-exercise cat /var/jenkins_home/secrets/initialAdminPassword
```

3. Open `http://localhost:8080` in your browser.
4. Complete the setup wizard (install suggested plugins, create admin user).
5. Create a new **Pipeline** job called `hello-world`.
6. In the Pipeline section, select **Pipeline script** and enter:

```groovy
pipeline {
    agent any

    stages {
        stage('Hello') {
            steps {
                echo 'Hello, Jenkins!'
                sh 'date'
                sh 'whoami'
                sh 'pwd'
            }
        }

        stage('System Info') {
            steps {
                sh 'java -version'
                sh 'cat /etc/os-release || echo "Not a Linux system"'
            }
        }
    }

    post {
        success {
            echo 'My first Jenkins pipeline succeeded!'
        }
    }
}
```

7. Click **Save** and then **Build Now**.
8. Click on build `#1` and view the **Console Output**.

**Expected outcome:** Both stages pass. You see the date, user, working directory, and Java version in the output.

---

### Exercise 2: Create a GitHub-Connected Pipeline

**Objective:** Connect Jenkins to a GitHub repository and build code from it.

**Steps:**

1. Create a new GitHub repository called `jenkins-practice`.

2. Clone the repository and create the following files:

`app.py`:
```python
def greet(name):
    return f"Hello, {name}! Welcome to Jenkins CI."

def add(a, b):
    return a + b

if __name__ == '__main__':
    print(greet("DevOps Engineer"))
    print(f"2 + 3 = {add(2, 3)}")
```

`test_app.py`:
```python
from app import greet, add

def test_greet():
    assert greet("World") == "Hello, World! Welcome to Jenkins CI."

def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
    assert add(0, 0) == 0
```

`requirements.txt`:
```
pytest==7.4.3
```

`Jenkinsfile`:
```groovy
pipeline {
    agent {
        docker { image 'python:3.11-slim' }
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'pip install -r requirements.txt'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'python -m pytest test_app.py -v'
            }
        }

        stage('Run Application') {
            steps {
                sh 'python app.py'
            }
        }
    }

    post {
        always {
            echo "Build result: ${currentBuild.currentResult}"
        }
    }
}
```

3. Push all files to GitHub:

```bash
git add .
git commit -m "Add Python app with Jenkinsfile"
git push origin main
```

4. In Jenkins, create a new **Pipeline** job called `jenkins-practice`.
5. In the Pipeline section:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: Your GitHub repo URL
   - Credentials: Your GitHub credentials
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
6. Click **Save** and **Build Now**.

**Expected outcome:** Jenkins clones the repo, installs pytest, runs tests (all pass), and executes the application.

---

### Exercise 3: Build and Push a Docker Image

**Objective:** Create a pipeline that builds a Docker image and pushes it to Docker Hub.

**Prerequisites:** Docker Hub account. Jenkins running with Docker socket mounted.

**Steps:**

1. In your `jenkins-practice` repository, add a `Dockerfile`:

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

2. Update the `Jenkinsfile`:

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'yourdockerhubusername/jenkins-practice'
        DOCKER_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh '''
                    pip install -r requirements.txt || true
                    python -m pytest test_app.py -v || echo "Running tests in Docker instead"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh "docker rmi ${DOCKER_IMAGE}:${DOCKER_TAG} || true"
                sh "docker rmi ${DOCKER_IMAGE}:latest || true"
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            cleanWs()
        }
        success {
            echo "Image pushed: ${DOCKER_IMAGE}:${DOCKER_TAG}"
        }
        failure {
            echo "Pipeline failed. Check the logs."
        }
    }
}
```

3. Add your Docker Hub credentials to Jenkins:
   - Go to **Manage Jenkins** > **Credentials** > **Add Credentials**
   - Kind: Username with password
   - ID: `dockerhub-credentials`
   - Username: Your Docker Hub username
   - Password: Your Docker Hub password or access token

4. Push changes and run the pipeline.

**Expected outcome:** Jenkins builds a Docker image, pushes it to Docker Hub, and cleans up local images. Verify by checking your Docker Hub repository.

---

### Exercise 4: Create a Multibranch Pipeline with Branch-Specific Behavior

**Objective:** Set up a Multibranch Pipeline that behaves differently for different branches.

**Steps:**

1. Update the `Jenkinsfile` in your `jenkins-practice` repo:

```groovy
pipeline {
    agent {
        docker { image 'python:3.11-slim' }
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install') {
            steps {
                sh 'pip install -r requirements.txt'
            }
        }

        stage('Lint') {
            steps {
                sh 'pip install flake8'
                sh 'flake8 app.py --max-line-length=120 || true'
            }
        }

        stage('Test') {
            steps {
                sh 'python -m pytest test_app.py -v'
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                echo 'Deploying to STAGING environment...'
                echo 'In a real project, this would deploy to a staging server.'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                echo 'Deploying to PRODUCTION environment...'
                echo 'In a real project, this would deploy to production.'
            }
        }
    }

    post {
        always {
            echo "Branch: ${env.BRANCH_NAME}, Result: ${currentBuild.currentResult}"
        }
    }
}
```

2. Push to `main`:

```bash
git add Jenkinsfile
git commit -m "Add branch-specific pipeline stages"
git push origin main
```

3. Create and push a `develop` branch:

```bash
git checkout -b develop
git push -u origin develop
```

4. Create and push a feature branch:

```bash
git checkout -b feature/new-endpoint
# Make some changes to app.py (add a new function)
git add .
git commit -m "Add new endpoint"
git push -u origin feature/new-endpoint
```

5. In Jenkins, create a **Multibranch Pipeline** job:
   - Name: `jenkins-practice-multibranch`
   - Branch Source: GitHub
   - Repository URL: Your repo URL
   - Credentials: Your GitHub credentials
   - Behaviors: Discover branches (all branches)

6. Click **Save** and let Jenkins scan the repository.

**Expected outcome:** Jenkins discovers all three branches (`main`, `develop`, `feature/new-endpoint`) and creates pipeline jobs for each. The `main` branch shows the "Deploy to Production" stage, the `develop` branch shows "Deploy to Staging", and the feature branch skips both deploy stages.

---

### Exercise 5: Build a Complete CI Pipeline with Parallel Testing and Notifications

**Objective:** Create a production-realistic CI pipeline that uses parallel stages, parameterized builds, and comprehensive error handling.

**Steps:**

1. Expand your project with the following structure:

```
jenkins-practice/
├── app.py
├── test_app.py
├── tests/
│   ├── test_unit.py
│   └── test_integration.py
├── requirements.txt
├── Dockerfile
├── Jenkinsfile
└── .gitignore
```

`tests/test_unit.py`:
```python
from app import greet, add

def test_greet_basic():
    assert greet("Alice") == "Hello, Alice! Welcome to Jenkins CI."

def test_greet_empty():
    assert greet("") == "Hello, ! Welcome to Jenkins CI."

def test_add_positive():
    assert add(1, 2) == 3

def test_add_negative():
    assert add(-5, 3) == -2

def test_add_zero():
    assert add(0, 0) == 0
```

`tests/test_integration.py`:
```python
import subprocess
import sys

def test_app_runs():
    result = subprocess.run(
        [sys.executable, 'app.py'],
        capture_output=True,
        text=True,
        timeout=5
    )
    assert 'Hello' in result.stdout
    assert result.returncode == 0
```

2. Create the comprehensive `Jenkinsfile`:

```groovy
pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Skip the test stage'
        )
        choice(
            name: 'LOG_LEVEL',
            choices: ['INFO', 'DEBUG', 'WARNING'],
            description: 'Application log level'
        )
    }

    environment {
        APP_NAME = 'jenkins-practice'
        DOCKER_IMAGE = "yourusername/${APP_NAME}"
    }

    options {
        timeout(time: 20, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'echo "Commit: $(git rev-parse --short HEAD)"'
                sh 'echo "Branch: ${GIT_BRANCH}"'
            }
        }

        stage('Setup') {
            steps {
                sh '''
                    python3 -m venv venv || true
                    . venv/bin/activate || true
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Quality Checks') {
            when {
                expression { return !params.SKIP_TESTS }
            }
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'python3 -m pytest tests/test_unit.py -v'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'python3 -m pytest tests/test_integration.py -v'
                    }
                }
                stage('Lint Check') {
                    steps {
                        sh '''
                            pip install flake8 || true
                            flake8 app.py --max-line-length=120 --statistics || echo "Lint warnings found"
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build \
                        --build-arg LOG_LEVEL=${params.LOG_LEVEL} \
                        -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                        -t ${DOCKER_IMAGE}:latest \
                        .
                """
                echo "Built image: ${DOCKER_IMAGE}:${BUILD_NUMBER}"
            }
        }

        stage('Test Docker Image') {
            steps {
                sh """
                    docker run --rm ${DOCKER_IMAGE}:${BUILD_NUMBER} python -c "from app import greet; print(greet('Docker'))"
                """
            }
        }

        stage('Push Image') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }
    }

    post {
        always {
            echo """
            ==============================
            Pipeline Summary
            ==============================
            Job:     ${env.JOB_NAME}
            Build:   #${env.BUILD_NUMBER}
            Status:  ${currentBuild.currentResult}
            Duration: ${currentBuild.durationString}
            URL:     ${env.BUILD_URL}
            ==============================
            """
        }

        success {
            echo 'All stages passed successfully.'
        }

        failure {
            echo 'Pipeline FAILED. Review the console output above for errors.'
        }

        cleanup {
            sh "docker rmi ${DOCKER_IMAGE}:${BUILD_NUMBER} || true"
            sh "docker rmi ${DOCKER_IMAGE}:latest || true"
            sh 'docker logout || true'
            cleanWs()
        }
    }
}
```

3. Push everything and run the pipeline:

```bash
git add .
git commit -m "Add comprehensive CI pipeline with parallel testing"
git push origin main
```

4. In Jenkins, trigger the build with **Build with Parameters**:
   - Try with `SKIP_TESTS = false` (default) -- all tests run.
   - Try with `SKIP_TESTS = true` -- tests are skipped.
   - Try different `LOG_LEVEL` values.

5. Intentionally break a test to see how Jenkins reports failures:
   - Change an assertion in `tests/test_unit.py` to something incorrect.
   - Push and observe the pipeline fail at the test stage.
   - Fix the test and push again.

**Expected outcome:** You have a fully functional CI pipeline with parallel testing, parameterized builds, Docker image building, conditional pushing, and comprehensive post-build reporting. You understand how to debug failures by reading console output and how branch-based conditions control the pipeline flow.

---

## Next Steps

After completing these exercises, you are ready to:

- Explore **Jenkins Shared Libraries** for reusable pipeline code across multiple projects.
- Set up **Jenkins agents** on separate machines for distributed builds.
- Integrate Jenkins with **Kubernetes** to dynamically provision build agents as pods.
- Add **Slack or email notifications** to your pipelines.
- Implement **deployment pipelines** that push to staging and production environments.
- Learn **GitHub Actions** or **GitLab CI/CD** -- the concepts you learned here transfer directly.

Continue to the next sections of DevOps-101 to build on this foundation.
