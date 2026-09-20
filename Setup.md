# Complete Project Setup — Python System Monitoring CI/CD DevSecOps

## 1. Project objective

The project was based on the public application:

```text
https://github.com/Aj7Ay/Python-System-Monitoring
```

The application is a Python/Flask system-monitoring web application.

Our goal was to take that application and build a DevSecOps delivery pipeline around it.

The final architecture was:

```text
GitHub
   ↓
Jenkins Controller
   ↓
Jenkins Agent
   ↓
SonarQube
   ↓
Quality Gate
   ↓
OWASP Dependency-Check
   ↓
Trivy
   ↓
Docker Build
   ↓
Docker Hub
   ↓
App Server 1
   +
App Server 2
   ↓
Application Load Balancer
   ↓
Internet / Browser
```

---

# 2. AWS infrastructure

We created five EC2 instances.

```text
1. Jenkins Controller
2. Jenkins Agent
3. SonarQube
4. Application Server 1
5. Application Server 2
```

We intentionally kept the AWS networking simple for this demo.

We used the existing/default VPC and security-group setup instead of building a complicated production VPC.

---

# 3. Jenkins Controller

Purpose:

```text
Pipeline orchestration
```

Responsibilities:

* Jenkins UI
* Jenkins job
* credentials
* agent management
* pipeline control
* SonarQube Quality Gate waiting

---

# 4. Jenkins Agent

Purpose:

```text
Actual CI/build worker
```

It performed:

```text
Git checkout
SonarScanner
OWASP Dependency-Check
Trivy
Docker build
Docker image scan
Docker push
```

Connection:

```text
Jenkins Controller
        |
        | SSH
        v
Jenkins Agent
```

---

# 5. SonarQube server

We created a dedicated SonarQube EC2 instance.

Installed SonarQube using Docker:

```bash
docker run -d \
  --name sonarqube \
  -p 9000:9000 \
  sonarqube:community
```

We accessed:

```text
http://<SONARQUBE-IP>:9000
```

---

# 6. SonarQube Linux prerequisite

SonarQube required the Linux VM map-count setting.

We ran:

```bash
sudo sysctl -w vm.max_map_count=524288
```

and persisted the configuration.

---

# 7. SonarQube project

Jenkins created/analyzed:

```text
Project key:
python-system-monitoring

Project name:
Python-System-Monitoring
```

Sources:

```text
src
```

Tests:

```text
tests
```

Python version configured for analysis:

```text
3.9
```

---

# 8. SonarQube Quality Gate

The important design was:

```text
Code Analysis
      ↓
Quality Gate
      ↓
Continue pipeline only when gate succeeds
```

We used:

```groovy
waitForQualityGate abortPipeline: true
```

Successful result:

```text
Quality gate is 'OK'
```

This proved that Jenkins was actually waiting for SonarQube instead of simply firing the scanner and moving on.

---

# 9. OWASP Dependency-Check

Purpose:

```text
Scan application dependencies for known vulnerabilities
```

We configured:

```text
DP-Check
```

and the NVD API credential:

```text
nvd-api-key
```

Pipeline generated:

```text
dependency-check-report.xml
```

Jenkins then published the report.

---

# 10. Trivy filesystem scan

Purpose:

```text
Scan the source tree/filesystem
```

Pipeline command:

```bash
trivy fs \
  --format table \
  --output trivy-fs-report.txt \
  --timeout 10m \
  .
```

Report was archived in Jenkins.

---

# 11. Docker image build

The upstream project already had:

```text
build/Dockerfile
```

and:

```text
makefile
```

The Makefile uses variables:

```text
IMAGE_REG
IMAGE_REPO
IMAGE_TAG
```

We overrode them from Jenkins.

Final image format:

```text
docker.io/cheesepopcorn/python-system-monitoring:<BUILD_NUMBER>
```

Example:

```text
docker.io/cheesepopcorn/python-system-monitoring:16
```

---

# 12. Important Makefile lesson

The original Makefile defaults were for another Docker namespace.

Instead of editing the Makefile, Jenkins passed:

```bash
IMAGE_REG=docker.io
IMAGE_REPO=cheesepopcorn/python-system-monitoring
IMAGE_TAG=<BUILD_NUMBER>
```

This worked because the Makefile variables use:

```text
?=
```

Therefore command-line values override defaults.

---

# 13. Docker image security scan

After building the image we scanned it using:

```bash
trivy image \
  --format table \
  --output trivy-image-report.txt \
  --timeout 10m \
  "$DOCKER_IMAGE:$IMAGE_TAG"
```

This creates a second security layer.

So we had:

```text
Source/filesystem scan
        +
Container image scan
```

---

# 14. Docker Hub registry

We created:

```text
cheesepopcorn/python-system-monitoring
```

in Docker Hub.

Jenkins authenticated using:

```text
dockerhub-creds
```

with a Docker Hub PAT.

Then:

```bash
docker push \
  docker.io/cheesepopcorn/python-system-monitoring:<BUILD_NUMBER>
```

---

# 15. Application Server 1

We launched the first application EC2.

Installed Docker.

Then pulled the image produced by Jenkins.

Example:

```bash
docker pull cheesepopcorn/python-system-monitoring:16
```

---

# 16. Run application container on App Server 1

We ran:

```bash
docker run -d \
  --name python-system-monitoring \
  -p 5000:5000 \
  --restart unless-stopped \
  cheesepopcorn/python-system-monitoring:16
```

Then verified:

```bash
docker ps
```

Container showed:

```text
0.0.0.0:5000->5000/tcp
```

---

# 17. Test App Server 1

We tested the application locally:

```bash
curl http://localhost:5000/
```

It returned the Flask HTML page.

Container logs showed Gunicorn:

```text
Starting gunicorn
Listening at:
http://0.0.0.0:5000
```

Therefore:

```text
App Server 1 = working
```

---

# 18. Application Server 2

We repeated the same deployment process on the second application server.

Installed Docker.

Pulled:

```text
cheesepopcorn/python-system-monitoring:16
```

Ran:

```bash
docker run -d \
  --name python-system-monitoring \
  -p 5000:5000 \
  --restart unless-stopped \
  cheesepopcorn/python-system-monitoring:16
```

Verified:

```bash
docker ps
curl http://localhost:5000/
```

Therefore:

```text
App Server 2 = working
```

---

# 19. Why two application servers?

We wanted the application layer to look like:

```text
             Application
             /          \
            /            \
       App Server 1   App Server 2
```

This gives us multiple backend instances behind a load balancer.

---

# 20. Create target group

AWS:

```text
EC2
  ↓
Target Groups
  ↓
Create target group
```

Configuration:

```text
Target type:
Instances

Name:
python-app-tg

Protocol:
HTTP

Port:
5000

IP type:
IPv4
```

---

# 21. Health check configuration

Health check:

```text
Protocol:
HTTP

Port:
Traffic port

Path:
/
```

This means AWS checks:

```text
http://<INSTANCE>:5000/
```

to determine whether the backend is usable.

---

# 22. Register application servers

Registered:

```text
App Server 1
App Server 2
```

both on:

```text
Port 5000
```

---

# 23. Create Application Load Balancer

AWS:

```text
EC2
  ↓
Load Balancers
  ↓
Create Load Balancer
  ↓
Application Load Balancer
```

Configuration:

```text
Name:
python-app-alb

Scheme:
Internet-facing

IP:
IPv4
```

Used the existing VPC.

Selected at least two subnets/AZs.

---

# 24. ALB security group

Created a dedicated security group for the ALB.

Inbound:

```text
HTTP
TCP
80
0.0.0.0/0
```

This makes the ALB the public entry point.

---

# 25. Application security-group rule

The application servers need port:

```text
5000
```

accessible from the ALB.

So the application security group should allow:

```text
TCP 5000
Source:
ALB security group
```

instead of exposing port 5000 to the whole internet.

---

# 26. ALB listener

Listener:

```text
HTTP :80
```

Default action:

```text
Forward to:
python-app-tg
```

So traffic flow becomes:

```text
Browser
   |
   | HTTP :80
   v
Application Load Balancer
   |
   | HTTP :5000
   +-------------------+
   |                   |
   v                   v
App Server 1       App Server 2
   |                   |
   +--------+----------+
            |
        Flask app
```

---

# 27. Target health

Before the ALB existed, the target group showed:

```text
0 Healthy
2 Unhealthy
```

That wasn't the final state because it wasn't associated with a load balancer yet.

After associating it with the ALB and fixing the security-group access, the application targets became usable by the load balancer.

---

# 28. Final application verification

After creating the ALB we took its DNS name.

Then opened:

```text
http://<ALB-DNS-NAME>
```

in a browser.

The website loaded successfully.

This was the final proof that the architecture worked end-to-end.

---

# 29. Final end-to-end flow

The complete project we built was:

```text
Developer
    |
    v
GitHub
    |
    v
Jenkins Controller
    |
    | SSH
    v
Jenkins Agent
    |
    +--> SonarQube
    |       |
    |       +--> Quality Gate
    |
    +--> OWASP Dependency-Check
    |
    +--> Trivy Filesystem Scan
    |
    +--> Docker Build
    |
    +--> Trivy Image Scan
    |
    +--> Docker Hub Push
                |
                v
        Docker image
                |
         +------+------+
         |             |
         v             v
    App Server 1   App Server 2
       :5000          :5000
         |             |
         +------+------+
                |
                v
      AWS Application Load Balancer
                |
                v
             Browser
                |
                v
          LIVE WEBSITE
```

---

# 30. What we actually completed

```text
AWS EC2 infrastructure              ✅
Jenkins Controller                   ✅
Jenkins Agent                        ✅
SSH agent connection                 ✅
Docker build worker                  ✅
SonarQube                            ✅
SonarQube Quality Gate               ✅
OWASP Dependency-Check               ✅
Trivy filesystem scan                ✅
Docker image build                   ✅
Trivy image scan                     ✅
Docker Hub push                      ✅
App Server 1                         ✅
App Server 2                         ✅
Dockerized application               ✅
ALB                                  ✅
Target Group                         ✅
Health checks                        ✅
Public website access                ✅
GitHub portfolio repo                ✅
```

---

# 31. What is NOT finished yet

This is important to remember.

### Automatic deployment

Currently:

```text
Jenkins
  ↓
Docker Hub
```

is automated.

But:

```text
Docker Hub
  ↓
App Server 1 / 2
```

was manually deployed.

So the next improvement is:

```text
Jenkins
  ↓
Docker Hub
  ↓
Deploy to App Server 1
  ↓
Health check
  ↓
Deploy to App Server 2
```

---

# 32. Other improvements we intentionally postponed

### Tests/lint

The original repository has testing/lint functionality, but our Jenkins pipeline did not yet add explicit:

```text
make test
make lint
```

stages.

This is a logical next CI improvement.

### Docker base-image modernization

The upstream Dockerfile currently uses:

```text
python:3.9-slim-buster
```

Trivy warned that Debian 10/Buster is no longer supported.

We intentionally left that alone while getting the complete pipeline working.

### Fully automated deployment

Not implemented yet.

### HTTPS

The ALB currently exposes HTTP.

HTTPS with an ACM certificate can be added later.

---

# 33. The project in one sentence

A useful interview description is:

> Built a Jenkins-based DevSecOps pipeline for a Python Flask application, integrating SonarQube Quality Gates, OWASP Dependency-Check, Trivy filesystem and image scanning, Docker Hub image publishing, and deployment of the containerized application across two AWS EC2 instances behind an Application Load Balancer.

---

# 34. The mental model to remember

Remember the project in five layers:

```text
1. SOURCE
GitHub

2. CI/CD ORCHESTRATION
Jenkins

3. SECURITY
SonarQube
OWASP
Trivy

4. CONTAINER
Docker
Docker Hub

5. RUNTIME
EC2 × 2
ALB
```

Or even simpler:

```text
GitHub
   ↓
Build
   ↓
Scan
   ↓
Package
   ↓
Push
   ↓
Deploy
   ↓
Load Balance
   ↓
Users
```
