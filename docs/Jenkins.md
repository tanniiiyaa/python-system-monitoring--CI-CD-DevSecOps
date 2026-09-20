# Jenkins Setup — Python System Monitoring DevSecOps

## 1. Jenkins architecture we used

We did **not** run everything on one Jenkins machine.

We used:

```text
Jenkins Controller
        |
        | SSH
        v
Jenkins Agent / Build Worker
```

### Jenkins Controller

Responsible for:

* Jenkins UI
* Creating/configuring jobs
* Pipeline orchestration
* Credentials
* Plugins
* Calling stages
* Waiting for SonarQube Quality Gate

### Jenkins Agent

Responsible for:

* Git checkout
* SonarScanner execution
* OWASP Dependency-Check
* Trivy
* Docker build
* Docker image scan
* Docker Hub push

The agent was given the label:

```text
docker-agent
```

The agent had:

```text
1 executor
Remote root directory:
 /home/ubuntu/jenkins-agent
```

---

# 2. Jenkins Controller EC2 setup

We launched a dedicated EC2 instance for Jenkins Controller.

After connecting to the server, we installed Jenkins and opened:

```text
http://<JENKINS-IP>:8080
```

We completed the normal Jenkins initial setup:

1. Started Jenkins.
2. Opened Jenkins in browser.
3. Unlocked Jenkins using the initial admin password.
4. Completed setup.
5. Logged into Jenkins.
6. Used Jenkins as the orchestration/controller machine.

---

# 3. Jenkins Agent EC2 setup

We launched a second EC2 for the Jenkins agent.

The important architecture decision was:

```text
Controller --> Agent private IP
```

NOT:

```text
Controller --> Agent public IP
```

The agent was in the same VPC/AZ environment, so we used the agent's **private IP** for the Jenkins SSH connection.

---

# 4. Create SSH access from Jenkins Controller to Jenkins Agent

We wanted Jenkins to connect to the agent using SSH.

We used the existing AWS:

```text
dev
```

key.

Initially we checked whether the PEM key already existed on the controller.

We searched using:

```bash
find /home /root -type f \( -name "*.pem" -o -name "*.key" \) 2>/dev/null
```

It wasn't there.

So we made the `dev.pem` key available on the Jenkins Controller and placed it under the Jenkins user's SSH area.

The SSH key was ultimately accessible to Jenkins.

---

# 5. Jenkins SSH credential

In Jenkins:

```text
Manage Jenkins
    ↓
Credentials
    ↓
Global credentials
```

Created an SSH credential.

Configuration:

```text
Kind:
SSH Username with private key

Username:
ubuntu

Private Key:
dev.pem

Credential ID:
dev-ssh
```

The important point:

Jenkins connects to the Linux agent as:

```text
ubuntu
```

---

# 6. Configure Jenkins node / agent

Go to:

```text
Manage Jenkins
    ↓
Nodes
    ↓
New Node
```

Created the agent:

```text
jenkins-agent-1
```

Configuration:

```text
Remote root directory:
/home/ubuntu/jenkins-agent

Labels:
docker-agent

Executors:
1
```

Launch method:

```text
Launch agents via SSH
```

Host:

```text
<Agent PRIVATE IP>
```

Credentials:

```text
dev-ssh
```

Host key verification:

```text
Known hosts file Verification Strategy
```

---

# 7. Verify Jenkins SSH connection

Before doing any pipeline work, we manually verified that the Jenkins service account could SSH to the agent.

We tested from the controller using:

```bash
sudo -u jenkins ssh -i /var/lib/jenkins/.ssh/dev.pem ubuntu@<AGENT-PRIVATE-IP>
```

Once that worked, Jenkins was able to launch the agent.

Result:

```text
Agent online
```

---

# 8. Install required tools on Jenkins Agent

The agent became our actual build worker.

We installed:

```text
Java 21
Git
Make
Docker
Trivy
```

We verified versions.

Examples:

```bash
java -version
git --version
make --version
docker --version
trivy --version
```

---

# 9. Fix Docker permission problem on Jenkins Agent

Initially Docker did not work for the `ubuntu` user.

We got:

```text
permission denied while trying to connect to the Docker API
```

The correct fix we used was:

```bash
sudo usermod -aG docker ubuntu
```

Then we restarted/reconnected the Jenkins agent session.

We deliberately **did not** use:

```bash
chmod 777 /var/run/docker.sock
```

because we wanted a cleaner permission model.

After reconnecting, Docker worked for the agent.

---

# 10. Create an agent test job

Before building the actual pipeline, we created a Freestyle job:

```text
agent-test
```

Configured it to run on:

```text
docker-agent
```

The job printed information such as:

```text
hostname
whoami
java -version
git --version
make --version
docker --version
trivy --version
```

This confirmed the Jenkins agent was really executing the work.

Result:

```text
Build successful
```

---

# 11. Install Jenkins plugins

We installed the plugins/components needed for the pipeline.

Important Jenkins functionality we configured:

### SonarQube

Used for:

* static code analysis
* Quality Gate

### OWASP Dependency-Check

Used for:

* dependency vulnerability scanning

### Pipeline support

Used for:

* Declarative Jenkinsfile pipeline

---

# 12. Configure SonarScanner tool in Jenkins

Go to:

```text
Manage Jenkins
    ↓
Tools
```

Configured SonarScanner:

```text
Name:
sonar-scanner
```

This name is important because the Jenkinsfile uses:

```groovy
SCANNER_HOME = tool 'sonar-scanner'
```

---

# 13. SonarQube token

On the SonarQube server we created a token.

Then in Jenkins:

```text
Manage Jenkins
    ↓
Credentials
    ↓
Global credentials
```

Created a secret credential.

Credential ID:

```text
sonar-token
```

Important lesson:

Our first SonarQube credential was created with the wrong scope:

```text
System
```

The pipeline couldn't access it.

We fixed this by creating/using the credential under:

```text
Global credentials (unrestricted)
```

So the pipeline could access:

```text
sonar-token
```

---

# 14. Configure SonarQube server in Jenkins

Go to:

```text
Manage Jenkins
    ↓
System
    ↓
SonarQube installations
```

Created:

```text
Name:
sonar-server

Server URL:
http://<SONARQUBE-IP>:9000

Authentication token:
sonar-token
```

The Jenkinsfile references:

```groovy
withSonarQubeEnv(
    installationName: 'sonar-server',
    credentialsId: 'sonar-token'
)
```

---

# 15. Configure SonarQube webhook

On SonarQube:

```text
Administration
    ↓
Configuration
    ↓
Webhooks
```

Created a Jenkins webhook:

```text
http://<JENKINS-IP>:8080/sonarqube-webhook/
```

Important:

The trailing `/` was included.

This allowed Jenkins':

```text
waitForQualityGate
```

to know when SonarQube had completed processing.

---

# 16. SonarQube pipeline configuration

Our scanner command became:

```bash
"$SCANNER_HOME/bin/sonar-scanner" \
  -Dsonar.projectKey=python-system-monitoring \
  -Dsonar.projectName=Python-System-Monitoring \
  -Dsonar.sources=src \
  -Dsonar.tests=tests \
  -Dsonar.python.version=3.9
```

Successful result:

```text
ANALYSIS SUCCESSFUL
```

Then:

```text
Quality gate is 'OK'
```

So SonarQube integration was completely validated.

---

# 17. Configure OWASP Dependency-Check

In:

```text
Manage Jenkins
    ↓
Tools
```

we configured:

```text
DP-Check
```

as the Dependency-Check installation.

---

# 18. NVD API key for OWASP

The first Dependency-Check run failed because the NVD API key was empty.

Error was essentially:

```text
Invalid API Key
length of 0 too short
```

So we created an NVD API key externally and stored it in Jenkins as a secret.

Jenkins credential:

```text
ID:
nvd-api-key
```

Credential scope:

```text
Global
```

We never put the key directly into the Jenkinsfile.

---

# 19. OWASP pipeline step

We used:

```groovy
dependencyCheck(
    odcInstallation: 'DP-Check',
    nvdCredentialsId: 'nvd-api-key',
    additionalArguments: '--scan ./ --format XML'
)
```

Then published the generated report:

```groovy
dependencyCheckPublisher(
    pattern: '**/dependency-check-report.xml',
    skipNoReportFiles: false,
    stopBuild: false
)
```

This worked successfully.

We saw:

```text
Writing XML report to:
dependency-check-report.xml
```

and Jenkins parsed the report.

---

# 20. Install Trivy on Jenkins Agent

Trivy runs on the agent because the agent does the security scanning.

We installed Trivy and verified:

```bash
trivy --version
```

---

# 21. Trivy filesystem scan

We added:

```groovy
stage('Trivy Filesystem Scan') {
    steps {
        sh '''
            trivy fs \
              --format table \
              --output trivy-fs-report.txt \
              --timeout 10m \
              .
        '''
    }
}
```

Then archived the report:

```groovy
archiveArtifacts(
    artifacts: 'trivy-fs-report.txt',
    fingerprint: true
)
```

This was successful.

---

# 22. Understand Trivy warning

Trivy printed:

```text
Unable to find python site-packages directory.
License detection is skipped.
```

This was only a warning.

The scan itself completed successfully.

---

# 23. Docker image configuration

The original repository Makefile contained:

```make
IMAGE_REG ?= docker.io
IMAGE_REPO ?= sevenajay/python-system-monitoring
IMAGE_TAG ?= latest
```

The important lesson:

Because the Makefile uses:

```text
?=
```

we could override these values from Jenkins without modifying the upstream Makefile.

---

# 24. Our Docker Hub repository

We created our own Docker Hub repository:

```text
cheesepopcorn/python-system-monitoring
```

Therefore Jenkins uses:

```groovy
DOCKER_REGISTRY = 'docker.io'
DOCKER_REPO = 'cheesepopcorn/python-system-monitoring'
DOCKER_IMAGE = 'docker.io/cheesepopcorn/python-system-monitoring'
IMAGE_TAG = "${BUILD_NUMBER}"
```

---

# 25. Docker Hub Jenkins credential

In Jenkins:

```text
Manage Jenkins
    ↓
Credentials
    ↓
Global
```

created:

```text
Credential ID:
dockerhub-creds

Kind:
Username with password
```

Username:

```text
cheesepopcorn
```

Password:

```text
Docker Hub Personal Access Token
```

Important:

The PAT is stored in Jenkins, never in Git.

---

# 26. First Docker Hub authentication failure

Initially Jenkins reported:

```text
incorrect username or password
```

We tested authentication manually.

Eventually Jenkins showed:

```text
Login Succeeded
```

So authentication was fixed.

---

# 27. First Docker Hub repository-name failure

After login worked, push still failed with:

```text
insufficient_scope
```

The reason was a typo in the repository name.

We initially used:

```text
chessepopcorn
```

Correct username was:

```text
cheesepopcorn
```

Correct repository:

```text
cheesepopcorn/python-system-monitoring
```

---

# 28. Second Docker pipeline mistake

Docker build was producing:

```text
docker.io/cheesepopcorn/python-system-monitoring:15
```

but Trivy was still scanning:

```text
docker.io/chessepopcorn/python-system-monitoring:15
```

So Trivy said:

```text
No such image
```

We fixed:

```groovy
DOCKER_IMAGE
```

to use:

```text
cheesepopcorn
```

After that the build and image scan worked.

---

# 29. Final Docker build command

Jenkins ultimately ran:

```groovy
make image \
  IMAGE_REG="$DOCKER_REGISTRY" \
  IMAGE_REPO="$DOCKER_REPO" \
  IMAGE_TAG="$IMAGE_TAG"
```

Result:

```text
docker.io/cheesepopcorn/python-system-monitoring:<BUILD_NUMBER>
```

---

# 30. Trivy Docker image scan

We added:

```groovy
trivy image \
  --format table \
  --output trivy-image-report.txt \
  --timeout 10m \
  "$DOCKER_IMAGE:$IMAGE_TAG"
```

Then:

```groovy
archiveArtifacts(
    artifacts: 'trivy-image-report.txt',
    fingerprint: true
)
```

This worked successfully.

---

# 31. Docker push stage

Final working authentication/push logic:

```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'dockerhub-creds',
        usernameVariable: 'DOCKER_USERNAME',
        passwordVariable: 'DOCKER_PASSWORD'
    )
]) {
    sh '''
        set +x

        echo "$DOCKER_PASSWORD" | docker login \
          --username "$DOCKER_USERNAME" \
          --password-stdin

        make push \
          IMAGE_REG="$DOCKER_REGISTRY" \
          IMAGE_REPO="$DOCKER_REPO" \
          IMAGE_TAG="$IMAGE_TAG"

        docker logout
    '''
}
```

Important security idea:

```text
Secrets -> Jenkins Credentials
```

not:

```text
Secrets -> Jenkinsfile / GitHub
```

---

# 32. Final Jenkins pipeline order

The Jenkins pipeline we successfully validated is:

```text
Clean Workspace
        ↓
Checkout
        ↓
SonarQube Analysis
        ↓
Quality Gate
        ↓
OWASP Dependency Check
        ↓
Publish OWASP Report
        ↓
Trivy Filesystem Scan
        ↓
Archive Trivy Report
        ↓
Docker Build
        ↓
Docker Image Scan
        ↓
Archive Trivy Image Report
        ↓
Docker Push
```

---

# 33. Final Jenkinsfile environment section

```groovy
environment {
    SCANNER_HOME = tool 'sonar-scanner'

    DOCKER_REGISTRY = 'docker.io'
    DOCKER_REPO = 'cheesepopcorn/python-system-monitoring'
    DOCKER_IMAGE = 'docker.io/cheesepopcorn/python-system-monitoring'
    IMAGE_TAG = "${BUILD_NUMBER}"
}
```

---

# 34. Jenkins setup final status

Completed and verified:

```text
Jenkins Controller             ✅
Jenkins Agent                  ✅
SSH Controller → Agent         ✅
Agent label docker-agent       ✅
Java                           ✅
Git                            ✅
Make                           ✅
Docker                         ✅
Trivy                          ✅
SonarQube integration          ✅
SonarQube Quality Gate         ✅
OWASP Dependency-Check         ✅
Trivy filesystem scan          ✅
Docker build                   ✅
Trivy image scan               ✅
Docker Hub login               ✅
Docker Hub image push          ✅
```

Not yet automated:

```text
Jenkins automatic deployment   ❌
```

At the time we stopped, deployment was still performed manually on the EC2 app servers.
