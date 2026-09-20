# Project Theory — CI/CD & DevSecOps

This is the theory/reference sheet for the Python System Monitoring DevSecOps project.

Application source:
https://github.com/Aj7Ay/Python-System-Monitoring

---

# 1. Project objective

The objective is to take a Python/Flask application and build a DevSecOps delivery process around it.

Main components:
- GitHub — source code
- Jenkins — CI/CD orchestration
- Jenkins Agent — build and security execution
- SonarQube — static code analysis
- SonarQube Quality Gate — quality control
- OWASP Dependency-Check — dependency vulnerability analysis
- Trivy — filesystem and container-image scanning
- Docker — containerization
- Docker Hub — image registry
- AWS EC2 — application runtime
- AWS Application Load Balancer — public traffic distribution

Overall lifecycle:
~~~text
Source → Build → Analyze → Scan → Package → Push → Deploy → Users
~~~

# 2. CI/CD theory

## Continuous Integration
CI means frequently integrating code into a shared repository and automatically validating each change.

Typical CI activities include source checkout, tests, static analysis, dependency checks, security scans, and artifact/image creation.

## Continuous Delivery / Deployment
CD takes a validated build toward the runtime environment. In this project the intended path is:
~~~text
Jenkins → Docker image → Docker Hub → EC2 application servers → Load Balancer → Users
~~~

# 3. DevSecOps theory

DevSecOps integrates security into the software delivery lifecycle rather than treating security as a separate final step.

~~~text
Build
 ↓
Code Quality
 ↓
Dependency Security
 ↓
Filesystem Security
 ↓
Container Security
 ↓
Publish
 ↓
Deploy
~~~

# 4. Jenkins Controller vs Agent

## Controller
The Controller provides the Jenkins UI, pipeline orchestration, job scheduling, credential/configuration management, and coordination of stages.

## Agent
The Agent performs the workload. Our agent handled Git, Java, Make, Docker, SonarScanner, OWASP Dependency-Check, Trivy, Docker build, image scan, and Docker push.

Architecture:
~~~text
Jenkins Controller
        | SSH
        v
Jenkins Agent (docker-agent)
        |
        v
Build / Scan / Package
~~~

# 5. Jenkins Credentials

Jenkins Credentials provide centralized secret storage.

Credentials used:

| ID | Purpose |
|---|---|
| dev-ssh | SSH access to Jenkins Agent |
| sonar-token | SonarQube authentication |
| nvd-api-key | NVD access for Dependency-Check |
| dockerhub-creds | Docker Hub authentication |

Secrets should remain in Jenkins credentials rather than in GitHub.

~~~text
GitHub → code/configuration
Jenkins Credentials → secrets
~~~

# 6. SonarQube theory

SonarQube performs static analysis of source code. It can identify code quality issues, maintainability problems, duplication, bugs, and security-related findings.

Flow:
~~~text
Checkout → SonarScanner → SonarQube → Quality Gate
~~~

## Quality Gate
A Quality Gate is a pipeline decision point. Jenkins waits for the SonarQube result and can stop the pipeline if the configured gate fails.

We used:
~~~groovy
waitForQualityGate abortPipeline: true
~~~

## Webhook
The SonarQube webhook lets SonarQube notify Jenkins when an analysis task has finished, allowing waitForQualityGate to continue.

# 7. OWASP Dependency-Check theory

Dependency-Check is a Software Composition Analysis tool. It analyzes third-party dependencies and compares them with known vulnerability information.

Flow:
~~~text
Application dependencies → Vulnerability data → Report
~~~

The Jenkins pipeline generated dependency-check-report.xml and published the result.

# 8. Trivy theory

Trivy was used at two different points.

## Filesystem scan
Scans the source workspace.

~~~bash
trivy fs .
~~~

## Image scan
Scans the final Docker image after it is built.

~~~text
Docker build → Trivy image scan
~~~

Using both gives source/filesystem visibility plus final-container visibility.

# 9. Docker theory

Docker packages an application and its runtime dependencies into a container image.

Our application listens on port 5000.

Container mapping:
~~~text
Host :5000 → Container :5000
~~~

Example runtime:
~~~bash
docker run -d --name python-system-monitoring -p 5000:5000 --restart unless-stopped cheesepopcorn/python-system-monitoring:<TAG>
~~~

# 10. Docker image tagging

We used the Jenkins build number as the Docker image tag.

Example:
~~~text
Build 14 → image :14
Build 15 → image :15
Build 16 → image :16
~~~

This gives every build an identifiable version instead of relying only on latest.

# 11. Docker Hub theory

Docker Hub acts as the container registry. Jenkins pushes the built image and application servers can pull the version they need.

~~~text
Jenkins Agent → docker build → docker push → Docker Hub
                                           ↓
                                     docker pull
                                           ↓
                                      EC2 servers
~~~

# 12. Makefile theory

The upstream application contains a Makefile with IMAGE_REG, IMAGE_REPO, and IMAGE_TAG variables.

Because these use default assignments, Jenkins can override them without modifying the upstream Makefile.

~~~bash
make image IMAGE_REG=docker.io IMAGE_REPO=cheesepopcorn/python-system-monitoring IMAGE_TAG=16
~~~

Important naming lesson:
~~~text
Registry = docker.io
Repository = cheesepopcorn/python-system-monitoring
Image = docker.io/cheesepopcorn/python-system-monitoring:16
~~~

Do not put the full image name into IMAGE_REPO when the Makefile already prepends IMAGE_REG.

# 13. AWS EC2 theory

EC2 provides virtual compute instances. We separated responsibilities across five instances:

~~~text
1. Jenkins Controller
2. Jenkins Agent
3. SonarQube
4. Application Server 1
5. Application Server 2
~~~

# 14. Why two application servers?

Two application servers allow the ALB to distribute requests between multiple backend targets and stop routing to a target that becomes unhealthy.

~~~text
Application Load Balancer
        /             \
       v               v
 App Server 1      App Server 2
    :5000             :5000
~~~

# 15. Application Load Balancer theory

An Application Load Balancer provides an HTTP/HTTPS entry point and forwards requests to registered targets.

Our flow:
~~~text
Internet → ALB :80 → Target Group → EC2 :5000
~~~

# 16. Target Group theory

The target group is the logical group of backend instances used by the ALB.

Configuration used:
- Name: python-app-tg
- Protocol: HTTP
- Port: 5000
- Health check path: /

Targets:
- App Server 1
- App Server 2

# 17. Health check theory

The ALB checks the application using HTTP on port 5000 and path /.

Conceptually:
~~~text
HTTP GET / :5000
       ↓
Healthy → can receive traffic
Unhealthy → removed from service
~~~

# 18. Security group theory

Security groups act as network firewalls.

Desired model:
~~~text
Internet
   | TCP 80
   v
ALB Security Group
   | TCP 5000
   v
Application Security Group
~~~

The application port should not need to be broadly exposed to the internet; the ALB should be the public entry point.

# 19. Why multiple security tools?

| Tool | Main focus |
|---|---|
| SonarQube | Source-code quality and static analysis |
| OWASP Dependency-Check | Third-party dependency vulnerabilities |
| Trivy filesystem | Workspace/filesystem scanning |
| Trivy image | Final container-image scanning |

This is defense in depth: each tool examines a different part of the delivery chain.

# 20. End-to-end architecture

~~~text
                         Internet
                            |
                            v
                    Application Load Balancer :80
                            |
                    +-------+-------+
                    |               |
                    v               v
              App Server 1    App Server 2
                 Docker           Docker
                 :5000            :5000
                    ^               ^
                    |               |
                    +-------+-------+
                            |
                       Docker Hub
                            ^
                            |
                       Jenkins Agent
                            ^
                            |
                     Jenkins Controller
                            ^
                            |
                          GitHub
~~~

# 21. Complete Jenkins pipeline theory

~~~text
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
Trivy Image Scan
      ↓
Archive Trivy Image Report
      ↓
Docker Hub Push
~~~

# 22. What was actually automated

The validated automation path was:

~~~text
GitHub → Jenkins → SonarQube → Quality Gate → OWASP → Trivy FS → Docker Build → Trivy Image → Docker Hub
~~~

# 23. What remained manual

The image deployment to App Server 1 and App Server 2 was manually performed after the successful Docker Hub push.

Therefore the remaining CD automation is:

~~~text
Docker Hub
   ↓
Deploy App Server 1
   ↓
Health check
   ↓
Deploy App Server 2
   ↓
ALB
~~~

# 24. Important troubleshooting lessons

## Credential scope
A pipeline credential created with an unsuitable scope may not be accessible from the pipeline. The working pipeline credentials were kept in Global credentials.

## Docker permissions
The Ubuntu agent user initially could not access Docker. The fix was to add the user to the docker group and reconnect the agent session.

~~~bash
sudo usermod -aG docker ubuntu
~~~

We did not solve this with chmod 777 on the Docker socket.

## Docker Hub naming
Authentication can succeed while a push fails when the repository namespace is wrong or the account lacks repository permission.

Correct namespace:
~~~text
cheesepopcorn/python-system-monitoring
~~~

## Image-name consistency
The Docker build, Trivy scan, and push stages must reference exactly the same registry/repository/tag.

# 25. Interview explanation

I implemented a Jenkins-based DevSecOps pipeline around a Python Flask system-monitoring application. The pipeline checks out the source from GitHub, performs SonarQube analysis with a Quality Gate, scans dependencies using OWASP Dependency-Check, scans the filesystem and resulting Docker image with Trivy, builds a versioned Docker image, and pushes it to Docker Hub. The application was deployed across two AWS EC2 instances behind an Application Load Balancer with health checks.

# 26. Memory shortcut

~~~text
SOURCE      → GitHub
ORCHESTRATE → Jenkins
QUALITY     → SonarQube
DEPENDENCY  → OWASP
SECURITY    → Trivy
PACKAGE     → Docker
STORE       → Docker Hub
RUN         → EC2
ROUTE       → ALB
~~~

Or simply:

~~~text
GitHub → Jenkins → Analyze → Scan → Build → Push → Deploy → Load Balance → Users
~~~