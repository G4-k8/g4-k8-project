# Kubernetes Manifest Deployment Order

This document outlines the exact order in which Kubernetes manifests should be applied to ensure proper resource creation and dependencies.

## Prerequisites

### Cluster Setup
Before applying any manifests, ensure your EKS cluster is created and ready:
```bash
# Create cluster using the provided configuration
eksctl create cluster -f eks_config.yaml

# Verify cluster is ready
kubectl cluster-info
kubectl get nodes
```

**Note**: The cluster will be named `clo835` as defined in `eks_config.yaml`.

## Required Configuration Changes

**IMPORTANT**: Before deploying, you must update the following values in the manifest files:

### 1. eks_config.yaml
```yaml
# Replace <AccountID> with your actual AWS account ID (12 digits)
iam:
  serviceRoleARN: arn:aws:iam::YOUR_ACCOUNT_ID:role/LabRole
managedNodeGroups:
  - name: nodegroup
    # ... other settings ...
    iam:
      instanceRoleARN: arn:aws:iam::YOUR_ACCOUNT_ID:role/LabRole
```

### 2. configmap.yaml
```yaml
data:
  IMAGE_URL: "s3://your-private-bucket/background.jpg"  # Replace with your S3 bucket URL
  USER_NAME: "Your Name"  # Replace with your actual name
```

### 3. secret.yaml
```yaml
data:
  DBUSER: cm9vdA==      # base64 for 'root' (current value)
  DBPWD: cHc=      # base64 for 'pw' (current value)

# To encode your own values:
# echo -n 'your_username' | base64
# echo -n 'your_password' | base64
```

### 4. serviceaccount.yaml
```yaml
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::YOUR_ACCOUNT_ID:role/YOUR_IRSA_ROLE  # Replace with your IRSA role ARN
```

### 5. web/flask-deploy.yaml
```yaml
spec:
  template:
    spec:
      containers:
      - name: flask-app
        image: <ECR_IMAGE_URL>  # Replace with your ECR image URL (e.g., 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest)
        env:
        - name: APP_COLOR
          value: blue  # Can be customized if needed
```

### 6. mysql/mysql-deploy.yaml
```yaml
spec:
  template:
    spec:
      containers:
      - name: mysql
        image: <ECR_MYSQL_IMAGE_URL>  # Replace with your MySQL ECR image URL
```

**Note**: The environment variable names in the Kubernetes manifests (`DBHOST`, `DBPORT`, `DBUSER`, `DBPWD`, `DATABASE`, `APP_COLOR`) match exactly with those used in the local Docker testing examples in `code/README.md` for consistency.

### 6. Required AWS Resources
Before deployment, ensure you have:
- **ECR Repository for Flask App**: Create and push your Flask app image
- **ECR Repository for MySQL**: Create and push your MySQL image (if using custom MySQL image)
- **S3 Bucket**: Private bucket with your background image
- **IAM Role**: For IRSA (IAM Roles for Service Accounts) with S3 access permissions
- **IAM Roles**: For EKS cluster and node groups (referenced in eks_config.yaml)

## Deployment Order

### 1. Namespace
```bash
kubectl apply -f namespace.yaml
```
- Creates the `final` namespace where all resources will be deployed

### 2. ConfigMap
```bash
kubectl apply -f configmap.yaml
```
- Provides background image URL and user name to the Flask application
- Must be created before Flask deployment
- ConfigMap name: `flask-config`

### 3. Secret
```bash
kubectl apply -f secret.yaml
```
- Stores MySQL username and password securely
- Must be created before MySQL and Flask deployments

### 4. PersistentVolumeClaim
```bash
kubectl apply -f mysql/pvc.yaml
```
- Creates 2Gi storage claim for MySQL database
- Must be created before MySQL deployment

# Install Driver to bound PVC
```bash
eksctl create addon --name aws-ebs-csi-driver --cluster newclo835 --service-account-role-arn arn:aws:iam::403539841960:role/AmazonEKS_EBS_CSI_DriverRole --force
```


### 5. ServiceAccount
```bash
kubectl apply -f serviceaccount.yaml
```
- Creates `clo835` service account with IRSA permissions for S3 access
- Must be created before Flask deployment

### 6. ClusterRole
```bash
kubectl apply -f role.yaml
```
- Creates `CLO835-users` cluster role with namespace permissions
- Must be created before RoleBinding

### 7. ClusterRoleBinding
```bash
kubectl apply -f rolebinding.yaml
```
- Binds the `CLO835-users` role to the `clo835` service account
- Must be created after ServiceAccount and ClusterRole

# Create secrets ecr to fetch images from ECR, use your Account ID
```bash
kubectl create secret docker-registry ecr-secret   --docker-server=403539841960.dkr.ecr.us-east-1.amazonaws.com   --docker-username=AWS   --docker-password="$(aws ecr get-login-password --region us-east-1)"   --namespace=final
```

### 8. MySQL StatefulSet
```bash
kubectl apply -f mysql/mysql-deploy.yaml
```
- Deploys MySQL database as a StatefulSet with 1 replica
- Uses PVC for persistent storage
- Uses Secret for credentials
- Uses ECR image for MySQL container
- Requires `ecr-secret` for ECR image pull authentication
- Maintains pod ordinality and stable network identity

### 9. MySQL Headless Service
```bash
kubectl apply -f mysql/mysql-service.yaml
```
- Creates a headless service for MySQL StatefulSet (ClusterIP: None)
- Exposes MySQL to Flask application via stable DNS
- Must be created after MySQL StatefulSet

### 10. Flask Application Deployment
```bash
kubectl apply -f web/flask-deploy.yaml
```
- Deploys Flask application with 1 replica
- Uses ECR image, ConfigMap, Secret, and ServiceAccount
- Listens on port 81
- Uses label `app: employees`
- Requires `ecr-secret` for ECR image pull authentication

### 11. Flask Service
```bash
kubectl apply -f web/flask-service.yaml
```
- Exposes Flask application to internet (LoadBalancer)
- Must be created after Flask deployment
- Selects pods with label `app: employees`

## Verification Commands

After deployment, verify resources:
```bash
# Check all resources in final namespace
kubectl get all -n final

# Check PVC status
kubectl get pvc -n final

# Check service endpoints
kubectl get svc -n final

# Check pod logs
kubectl logs -f deployment/flask-app -n final
kubectl logs -f deployment/mysql -n final
```

## Important Notes

- **Cluster**: Ensure the `clo835` cluster is running before applying manifests
- **Dependencies**: Each step depends on the previous steps being completed successfully
- **Namespace**: All resources (except ClusterRole and ClusterRoleBinding) are created in the `final` namespace
- **Secrets**: Never commit real secrets to Git. Use placeholders and update securely
- **IRSA**: Ensure the IAM role ARN in serviceaccount.yaml is correct for S3 access
- **ECR Image**: Update the image URL in flask-deploy.yaml with your actual ECR image
- **Account ID**: Update `<AccountID>` in eks_config.yaml with your actual AWS account ID
- **S3 Access**: Ensure your IRSA role has proper S3 permissions for the background image bucket
- **ECR Secret**: Ensure `ecr-secret` exists for ECR image pull authentication

## Troubleshooting

If deployment fails:
1. Check cluster status: `kubectl cluster-info`
2. Check resource status: `kubectl describe <resource> -n final`
3. Check pod logs: `kubectl logs <pod-name> -n final`
4. Verify ConfigMap and Secret values are correct
5. Ensure ECR image is accessible and exists
6. Verify S3 bucket permissions and image accessibility
7. Check if `ecr-secret` exists for ECR authentication

## Cleanup

To delete the entire cluster and all resources:
```bash
eksctl delete cluster -f eks_config.yaml
```

---

## **How to Fix**

### **1. Edit your MySQL deployment environment variables:**
- Remove the `MYSQL_USER` and `MYSQL_PASSWORD` environment variables.
- Only set `MYSQL_ROOT_PASSWORD` and (optionally) `MYSQL_DATABASE`.

**Example:**
```yaml
env:
  - name: MYSQL_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: DBPWD
  - name: MYSQL_DATABASE
    value: employees
```

---

### **2. Apply the changes:**
```bash
kubectl apply -f k8s-manifests/mysql/mysql-deploy.yaml
```

### **3. Delete the old pod to force recreation:**
```bash
kubectl delete pod -l app=mysql -n final
```

---

### **4. Check pod status and logs again:**
```bash
kubectl get pods -n final
kubectl logs <new-mysql-pod-name> -n final
```

---

## **Summary Table**

| Variable             | Use for root? | Use for regular user? | Your case |
|----------------------|:-------------:|:---------------------:|:---------:|
| MYSQL_ROOT_PASSWORD  |      ✅       |          ❌           |   YES     |
| MYSQL_USER           |      ❌       |          ✅           |   REMOVE  |
| MYSQL_PASSWORD       |      ❌       |          ✅           |   REMOVE  |

---

**Make this change and your MySQL pod should start successfully!**  
Let me know if you need the exact YAML edit or want to see the full corrected manifest.

## Testing the Deployed Flask App

Once all pods and services are running, you can test your Flask web application as follows:

1. **Get the External URL:**
   - Run:
     ```sh
     kubectl get svc -n final
     ```
   - Look for the `EXTERNAL-IP` or DNS name under the `flask-service` row. Example:
     ```
     service/flask-service    LoadBalancer   ...   ad94b1183e1bb421189332f88111dac7-221655876.us-east-1.elb.amazonaws.com   81:32652/TCP   ...
     ```
   - The external URL will be:
     ```
     http://<EXTERNAL-IP-or-DNS>:81
     ```
     Example:
     http://ad94b1183e1bb421189332f88111dac7-221655876.us-east-1.elb.amazonaws.com:81

2. **Open in Your Browser:**
   - Copy the URL and open it in your web browser to access the Flask app.

3. **Test with curl (optional):**
   - You can also test from the command line:
     ```sh
     curl http://<EXTERNAL-IP-or-DNS>:81
     ```

4. **What Should You See?**
   - You should see your Flask app’s homepage or the default route output.

5. **Troubleshooting:**
   - If the page doesn’t load:
     - Wait a few minutes for the LoadBalancer to become active.
     - Ensure your AWS security group allows inbound traffic on port 81.
     - Check pod logs for errors:
       ```sh
       kubectl logs <flask-pod-name> -n final
       ```
   - If you see errors, review the pod logs and service configuration.