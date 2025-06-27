# Node.js MySQL CRUD on AWS EKS

## Overview
This repository contains Kubernetes manifests and configuration for deploying a Node.js MySQL CRUD application on AWS EKS. The setup includes a static Node.js app, a MySQL database with persistence, horizontal scaling, health probes, secure secret management with AWS Secrets Manager, and a custom Docker image.

## Architecture Details
- **Presentation**: AWS EKS cluster with an Ingress (AWS Load Balancer) routing to a Node.js app.
- **Application**: Node.js app Deployment with HPA for scaling.
- **Data**: MySQL StatefulSet with EBS-backed persistence.
- **Security**: Secrets fetched via `SecretProviderClass` from AWS Secrets Manager.
- **Monitoring**: Liveness and readiness probes for health checks.

### Diagram
[Users] --> [AWS Load Balancer (Ingress)] --> [Node.js Service] --> [Node.js Pods]
|                                      |
+---------> [MySQL StatefulSet] --> [EBS PV/PVC]
|
+-----> [AWS Secrets Manager] <--> [SecretProviderClass]


## Key Components

### Dockerfile
- **Purpose**: Defines the Docker image for the Node.js application.
- **Configuration**:
  ```Dockerfile
  FROM node:16
  WORKDIR /app
  COPY . .
  RUN npm install
  CMD ["node", "app.js"]

# Explanation:
- Base Image: Uses node:16 for Node.js 16.x.
- WORKDIR: Sets /app as the working directory.
- COPY: Copies the application files from the GitHub repo.
- RUN: Installs dependencies via npm install.
- CMD: Starts the app with node app.js.
- Build Process: docker build -t yourusername/nodejs-mysql-crud . and docker push yourusername/nodejs-mysql-crud.
- Benefits: Ensures a consistent runtime environment; simplifies deployment.
- Drawbacks: Requires manual dependency updates; no multi-stage build for smaller images.


# Deployment
- Node.js App: Deployed as a Kubernetes Deployment with 2 replicas, using the custom Docker image.
- MySQL: Managed as a StatefulSet with a headless service, using EBS for persistent storage.
- Ingress: Configured with the AWS Load Balancer Controller for external access (e.g., api.yourdomain.com).
- Persistent Volume: EBS-backed PVC for MySQL data.

# Horizontal Pod Autoscaler (HPA)
- Purpose: Automatically scales the Node.js Deployment based on CPU usage.
- Configuration:
- minReplicas: 1, maxReplicas: 5.
- Scales when average CPU utilization exceeds 70%.
- Setup: Requires the Metrics Server (kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml).
- Benefits: Improves availability under load; reduces manual intervention.
- Drawbacks: Costs may increase; requires accurate CPU thresholds.

# Liveness and Readiness Probes
- Liveness Probe: Checks for "Users List" in the / endpoint response every 10 seconds, restarting the pod if unhealthy (after 3 failures, 30s delay).
- Readiness Probe: Verifies the app is ready to serve traffic every 5 seconds (5s delay).
- Implementation: Uses httpGet with exec for liveness to grep the response.
- Benefits: Ensures pod health and prevents traffic to unready instances.
- Drawbacks: May need tuning for complex apps.

# SecretProviderClass
- Purpose: Fetches MySQL credentials (mysql-root-password, mysql-user, mysql-password) from AWS Secrets Manager.
- Configuration: Uses the secrets-store-csi-driver to mount secrets as volumes, referenced via environment variables.
- Setup: Requires the driver installed on EKS and a secret named mysql-credentials in Secrets Manager.
- Benefits: Enhances security by avoiding hardcoded secrets; centralizes credential management.
- Drawbacks: Adds dependency on AWS and driver setup.

