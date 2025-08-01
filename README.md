# Web App Deployment on Amazon EKS

This project demonstrates a complete 2-tier containerized web application deployment on Amazon EKS (Elastic Kubernetes Service). The application consists of a Flask frontend that connects to a MySQL backend database.

## Project Overview

This project implements a full-stack web application with the following components:
- **Frontend**: Flask web application serving employee management functionality
- **Backend**: MySQL database for data persistence
- **Infrastructure**: Kubernetes manifests for container orchestration on AWS EKS
- **Storage**: Private S3 bucket integration for dynamic content
- **CI/CD**: GitHub Actions for automated container image building and deployment

## Key Features & Architecture

- **Containerized Deployment**: Both Flask and MySQL applications run in separate containers
- **Kubernetes Orchestration**: Complete manifest set for production-ready deployment
- **Persistent Storage**: MySQL data persisted using AWS EBS volumes and PersistentVolumeClaims
- **Secure Configuration**: Database credentials stored as Kubernetes Secrets
- **Dynamic Content**: Background images loaded from private S3 bucket via ConfigMap
- **Cloud Integration**: ServiceAccount with IAM roles for secure AWS service access
- **Load Balancing**: External access provided through Kubernetes LoadBalancer service
- **Scalability**: Horizontal Pod Autoscaler (HPA) for automatic scaling based on CPU usage

## AWS Services Integration

- **Amazon EKS**: Managed Kubernetes service for container orchestration
- **Amazon ECR**: Private container registry for storing application images
- **Amazon S3**: Object storage for dynamic web content (background images)
- **Amazon EBS**: Block storage for MySQL database persistence
- **AWS IAM**: Identity and access management for secure service integration

## Security Considerations

- Database credentials are base64 encoded and stored as Kubernetes Secrets
- AWS credentials are managed through temporary session tokens for educational environments
- ServiceAccount configured for IAM role assumption (IRSA pattern)
- Network isolation through Kubernetes namespaces and service selectors

## Getting Started

1. **Prerequisites**: AWS account with EKS access, kubectl, and eksctl installed
2. **Cluster Creation**: Deploy EKS cluster using the provided `eks_config.yaml`
3. **Application Deployment**: Apply Kubernetes manifests in the specified order
4. **Access**: Connect to the application via the LoadBalancer external URL

For detailed deployment instructions, refer to `k8s-manifests/DEPLOYMENT_ORDER.md`.