# Web App Deployment on Amazon EKS

This project is a 2-tier containerized web application consisting of a Flask frontend and MySQL backend, deployed on Amazon EKS using Kubernetes.

## Project Structure

- `code/` – Flask web application source code
- `k8s-manifests/` – Kubernetes manifests for deploying the application

## Key Features

- Flask application listens on port 81
- Background image loaded dynamically from a private S3 bucket (URL provided via ConfigMap)
- MySQL credentials stored securely as Kubernetes Secrets
- Application Docker image is built using GitHub Actions and published to Amazon ECR
- Application is exposed to the internet using a Kubernetes **LoadBalancer** service
- Persistent storage for MySQL using EBS volumes and PersistentVolumeClaim
- IRSA-enabled ServiceAccount for secure access to private S3 bucket

## Deployment Steps (Overview)

1. Build and push the Docker image to Amazon ECR
2. Create an Amazon EKS cluster with `eksctl`
3. Apply the Kubernetes manifests in `k8s-manifests/`
4. Access the app via the external LoadBalancer URL