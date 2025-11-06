# DevOps CI/CD Pipeline - Spring Boot & Node.js

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Java](https://img.shields.io/badge/Java-21-orange.svg)
![Node.js](https://img.shields.io/badge/Node.js-20-green.svg)
![Docker](https://img.shields.io/badge/Docker-Multistage-blue.svg)
![Kubernetes](https://img.shields.io/badge/Kubernetes-EKS-blue.svg)

A comprehensive DevOps CI/CD pipeline implementation featuring Spring Boot microservices backend, Node.js/React frontend, containerization with Docker, orchestration with Kubernetes on AWS EKS, automated testing, security scanning, and deployment through Jenkins.

## 📋 Table of Contents

- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Repository Structure](#repository-structure)
- [Pipeline Stages](#pipeline-stages)
- [Setup Instructions](#setup-instructions)
- [Docker Images](#docker-images)
- [Kubernetes Deployment](#kubernetes-deployment)
- [Secrets Management](#secrets-management)
- [Best Practices](#best-practices)
- [Contributing](#contributing)

## 🏗 Architecture

```
┌─────────────┐     ┌──────────────┐     ┌───────────────┐
│   GitHub    │────▶│   Jenkins    │────▶│  SonarQube    │
│  Repository │     │   Pipeline   │     │ Code Analysis │
└─────────────┘     └──────┬───────┘     └───────────────┘
                           │
                           ▼
                    ┌──────────────┐     ┌───────────────┐
                    │    Docker    │────▶│     Trivy     │
                    │    Build     │     │   Security    │
                    └──────┬───────┘     └───────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │    Nexus     │
                    │  Repository  │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   AWS EKS    │
                    │  Kubernetes  │
                    └──────────────┘
```

## 🛠 Tech Stack

### Backend
- **Spring Boot 3.x** - Java microservices framework
- **Java 21** - Programming language
- **Maven** - Build automation

### Frontend
- **Node.js 20** - JavaScript runtime
- **React** - Frontend framework
- **Nginx** - Web server for production

### DevOps Tools
- **Docker** - Containerization with multistage builds
- **Kubernetes** - Container orchestration on AWS EKS
- **Jenkins** - CI/CD automation
- **Nexus Repository** - Artifact and Docker image storage
- **SonarQube** - Code quality and security analysis
- **Trivy** - Container vulnerability scanning
- **AWS EKS** - Managed Kubernetes service

## 📁 Repository Structure

```
devops-pipeline-springboot-nodejs/
├── backend/
│   ├── Dockerfile              # Multistage Docker build for Spring Boot
│   ├── pom.xml                # Maven configuration
│   └── src/                   # Spring Boot source code
├── frontend/
│   ├── Dockerfile              # Multistage Docker build for React
│   ├── nginx.conf             # Nginx configuration
│   ├── package.json           # npm dependencies
│   └── src/                   # React source code
├── k8s/
│   ├── namespace.yaml         # Kubernetes namespace
│   ├── backend-deployment.yaml # Backend deployment & service
│   ├── frontend-deployment.yaml# Frontend deployment & service
│   ├── configmap.yaml         # Configuration maps
│   ├── secrets.yaml           # Secrets template
│   └── ingress.yaml           # Ingress configuration
├── Jenkinsfile                # Complete CI/CD pipeline
├── .gitignore                 # Git ignore patterns
├── LICENSE                    # MIT License
└── README.md                  # This file
```

## 🚀 Pipeline Stages

### 1. Checkout
- Pulls latest code from GitHub repository
- Captures git commit hash for version tracking

### 2. Build
- **Backend**: Maven builds Spring Boot application
- **Frontend**: npm builds React application
- Both stages run in parallel for efficiency

### 3. Unit Tests
- **Backend**: JUnit tests with Maven
- **Frontend**: Jest tests with coverage
- Test results published to Jenkins

### 4. Code Quality Analysis
- SonarQube scans for:
  - Code smells
  - Security vulnerabilities
  - Code coverage
  - Technical debt

### 5. Quality Gate
- Blocks deployment if quality standards not met
- Configurable thresholds for coverage and issues

### 6. Docker Image Build
- Multistage builds for both backend and frontend
- Minimal production images
- Versioned with build number

### 7. Security Scanning
- Trivy scans Docker images for:
  - Known CVEs
  - OS vulnerabilities
  - Application dependencies
- Build fails on CRITICAL vulnerabilities

### 8. Push to Nexus
- Docker images pushed to Nexus Docker registry
- Tagged with version and 'latest'

### 9. Deploy to EKS
- kubectl applies Kubernetes manifests
- Rolling updates for zero-downtime
- Automatic rollback on failure

### 10. Verification
- Validates pod health
- Checks service endpoints
- Displays deployment status

## ⚙️ Setup Instructions

### Prerequisites

1. **Jenkins Setup**
   ```bash
   # Install required plugins
   - Docker Pipeline
   - Kubernetes CLI
   - SonarQube Scanner
   - AWS CLI
   ```

2. **Nexus Repository**
   - Configure Docker registry on port 8082
   - Create Maven repository

3. **SonarQube**
   - Install and configure SonarQube server
   - Generate authentication token

4. **AWS EKS**
   ```bash
   # Create EKS cluster
   eksctl create cluster --name devops-cluster --region us-east-1
   
   # Configure kubectl
   aws eks update-kubeconfig --name devops-cluster --region us-east-1
   ```

5. **Trivy Installation**
   ```bash
   # Install Trivy on Jenkins agents
   wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
   echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
   sudo apt-get update
   sudo apt-get install trivy
   ```

### Jenkins Configuration

1. **Credentials**
   - `nexus-docker-credentials`: Nexus Docker registry credentials
   - `nexus-credentials`: Nexus Maven credentials
   - `sonarqube-token`: SonarQube authentication token
   - `aws-credentials`: AWS access key and secret

2. **Global Tools**
   - Maven 3.9+
   - JDK 21
   - Node.js 20
   - Docker

3. **Pipeline Setup**
   ```groovy
   // Create Pipeline job
   // Set SCM to Git: https://github.com/omkar-khot/devops-pipeline-springboot-nodejs
   // Script Path: Jenkinsfile
   ```

## 🐳 Docker Images

### Backend Dockerfile (Multistage)

**Build Stage**:
- Uses `maven:3.9.6-eclipse-temurin-21`
- Downloads dependencies (cached)
- Compiles application

**Runtime Stage**:
- Uses `eclipse-temurin:21-jre-alpine`
- Minimal image size (~200MB)
- Non-root user for security
- Health checks configured

### Frontend Dockerfile (Multistage)

**Build Stage**:
- Uses `node:20-alpine`
- Installs dependencies
- Builds production bundle

**Runtime Stage**:
- Uses `nginx:1.25-alpine`
- Serves static files
- Custom nginx configuration
- Minimal image size (~50MB)

## ☸️ Kubernetes Deployment

### Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: devops-app
```

### Backend Deployment
- 3 replicas for high availability
- Resource limits: 1Gi RAM, 500m CPU
- Liveness and readiness probes
- Rolling update strategy

### Frontend Deployment
- 2 replicas
- Resource limits: 256Mi RAM, 200m CPU
- Health checks configured

### Services
- Backend: ClusterIP on port 8080
- Frontend: LoadBalancer on port 80

### Ingress
- Path-based routing
- TLS termination (optional)

## 🔐 Secrets Management

### AWS Secrets Manager
```bash
# Create secret
aws secretsmanager create-secret \
    --name devops-app-secrets \
    --secret-string '{"database-url":"postgresql://..."}'
```

### Kubernetes Secrets
```bash
# Create from literal
kubectl create secret generic app-secrets \
    --from-literal=database-url=postgresql://... \
    -n devops-app
```

### Jenkins Credentials
- Stored in Jenkins credential store
- Referenced by ID in Jenkinsfile
- Never hardcoded in code

## 📊 Monitoring & Logging

- **Prometheus**: Metrics collection
- **Grafana**: Dashboards and visualization
- **ELK Stack**: Centralized logging
- **AWS CloudWatch**: Infrastructure monitoring

## ✅ Best Practices Implemented

1. **Security**
   - Non-root containers
   - Vulnerability scanning
   - Secrets management
   - Image signing (optional)

2. **Efficiency**
   - Multistage Docker builds
   - Parallel pipeline stages
   - Docker layer caching
   - Resource limits

3. **Reliability**
   - Health checks
   - Rolling updates
   - Automated testing
   - Quality gates

4. **Observability**
   - Structured logging
   - Metrics collection
   - Distributed tracing
   - Alerting

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 📧 Contact

**Omkar Khot**
- GitHub: [@omkar-khot](https://github.com/omkar-khot)

## 🙏 Acknowledgments

- Spring Boot community
- React community
- Kubernetes community
- Jenkins community
- DevOps best practices resources

---

**Note**: This is a template repository for demonstrating DevOps CI/CD pipeline concepts. Adapt configurations to your specific requirements before production use.
