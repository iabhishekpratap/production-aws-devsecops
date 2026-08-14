# Production-Grade AWS DevSecOps Platform

[![Last Commit](https://img.shields.io/github/last-commit/iabhishekpratap/production-aws-devsecops)]([https://github.com/USERNAME/REPOSITORY](https://github.com/iabhishekpratap/production-aws-devsecops))    [![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

 ---

## Overview

Production-grade AWS DevSecOps platform that automates infrastructure provisioning, secure CI/CD, GitOps-based EKS deployments, and Kubernetes observability using Terraform, Jenkins, Argo CD, Prometheus, and Grafana.

Solve a key challenge:

- Early-stage startups often struggle with manually managing cloud infrastructure, CI/CD, security checks, Kubernetes deployments, and monitoring, leading to operational overhead and deployment errors. 

- This project automates the complete DevSecOps workflow using Terraform, Jenkins, AWS EKS, ECR, Argo CD, SonarQube, Trivy, Prometheus, and Grafana for secure, repeatable, and production-ready application

---

## Architecture

![Three-Tier Banner](assets/Three-Tier.gif)

## Repository Structure

```markdown
/Application-Code
The Application-Code directory contains the source code for the Three-Tier Web Application. Dive into this directory to explore the frontend and backend implementations.

/Jenkins-Pipeline-Code
In the Jenkins-Pipeline-Code directory, you'll find Jenkins pipeline scripts. These scripts automate the CI/CD process, ensuring smooth integration and deployment of your application.

/Jenkins-Server-TF
Explore the Jenkins-Server-TF directory to find Terraform scripts for setting up the Jenkins Server on AWS. These scripts simplify the infrastructure provisioning process.

/Kubernetes-Manifests-files
The Kubernetes-Manifests-Files directory holds Kubernetes manifests for deploying your application on AWS EKS. Understand and customize these files to suit your project needs.

/assets
Contains Screenshots & Arch. Diagram.
```

## Project Details

 **Tools Explored:**

- Terraform & AWS CLI for AWS infrastructure
- Jenkins, Sonarqube, Terraform, Kubectl, and more for CI/CD setup
- Helm, Prometheus, and Grafana for Monitoring
- ArgoCD for GitOps practices

**High-Level Overview:**

- IAM User setup & Terraform magic on AWS
- Jenkins deployment with AWS integration
- EKS Cluster creation & Load Balancer configuration
- Private ECR repositories for secure image management
- Helm charts for efficient monitoring setup
- GitOps with ArgoCD - the cherry on top!

## Getting Started

To get started with this project, refer to our [Comprehensive Guide](https://github.com/iabhishekpratap/production-aws-devsecops/blob/main/docs/detailed-docs.md) that walks you through IAM user setup, infrastructure provisioning, CI/CD pipeline configuration, EKS cluster creation, and more.

## Contributing

We welcome contributions! If you have ideas for enhancements or find any issues, please open a pull request or file an issue.

Happy Coding! 🚀
