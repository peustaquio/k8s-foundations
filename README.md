k8s-foundations
A hands-on Kubernetes learning lab built with RHEL 9, Docker and Kind.

The main goal of this repository is not simply to deploy applications, but to understand how Kubernetes components work together, validate each layer independently, and practice troubleshooting in a real lab environment.

Overview
This lab was built incrementally.

The first phase focused on Kubernetes fundamentals using NGINX.

The second phase applied those concepts to a real application: a Simon memory game built with Flask and PostgreSQL, originally running with Docker Compose and later deployed on Kubernetes.

Phase 1 - Kubernetes Foundations
The first part of the lab focused on understanding the main Kubernetes building blocks.

Topics practiced:

Kind multi-node cluster
Control plane and worker nodes
Namespaces
Deployments
ReplicaSets
Pods
Services
ConfigMaps
Secrets
PersistentVolumeClaims
NGINX Ingress Controller
Host-based routing
Internal Kubernetes DNS
Container networking
Troubleshooting
Basic architecture
The initial NGINX lab followed this request path:

Notebook → Local DNS resolution → RHEL 9 → Docker → Kind → NGINX Ingress Controller → Ingress → Service → Pod

Ingress troubleshooting
One of the most valuable exercises in this lab was troubleshooting external access to the NGINX application.

The application itself was healthy, but external connections initially failed.

The troubleshooting process included validating:

Pod status
Deployment status
Service
Service endpoints
Ingress configuration
NGINX Ingress Controller configuration
Docker port mapping
Node placement
The root cause was related to topology.

Ports 80 and 443 were mapped through the Kind control-plane node, while the NGINX Ingress Controller was running on a worker node.

After scheduling the Ingress Controller on the control-plane node, connectivity progressed from:

Connection refused → HTTP 404 → Welcome to nginx

The HTTP 404 was an important clue because it demonstrated that traffic was finally reaching the Ingress Controller.

The final step was using the correct hostname configured in the Ingress rule.

The Windows hosts file provided local name resolution:

nginx-dev.local → RHEL 9 IP → Ingress → Service → NGINX Pod

This exercise reinforced an important troubleshooting principle:

Understand the complete request path first, then validate and eliminate one layer at a time.

Phase 2 - Simon Game on Kubernetes
After validating the Kubernetes fundamentals with NGINX, the next goal was to deploy an existing application.

The Simon Game was originally running through Docker Compose with two main components:

Flask application
PostgreSQL database
Docker Compose architecture:

Simon / Flask → PostgreSQL → Docker persistent volume

The application also contains a leaderboard stored in PostgreSQL.

Kubernetes architecture
The Docker Compose environment was migrated manually, component by component, instead of using an automated conversion tool.

The resulting architecture is:

Notebook → simon.local → RHEL 9 → Kind → NGINX Ingress Controller → Simon Ingress → Simon Service → Simon Pod → PostgreSQL Service → PostgreSQL Pod → PersistentVolumeClaim

Simon Kubernetes Components
Namespace
A dedicated namespace was created for the application:

simon

This keeps the Simon resources logically isolated from the other lab workloads.

PostgreSQL Secret
PostgreSQL credentials are provided to the containers using a Kubernetes Secret.

The application receives values such as:

DB_USER
DB_PASSWORD
PostgreSQL Persistent Storage
PostgreSQL uses a PersistentVolumeClaim so that database data is not tied directly to the lifecycle of the PostgreSQL Pod.

This separates:

Pod lifecycle from Data lifecycle

PostgreSQL Deployment
PostgreSQL runs as a Kubernetes workload using a Deployment.

The deployment uses the PostgreSQL container image and mounts persistent storage through the PVC.

PostgreSQL Service
A ClusterIP Service exposes PostgreSQL internally to the Kubernetes cluster.

Instead of connecting directly to a Pod IP, the Simon application uses:

DB_HOST=postgres

Kubernetes internal DNS resolves the Service name.

This means the application does not need to know the IP address of the PostgreSQL Pod.

Simon ConfigMap
Non-sensitive application configuration is stored in a ConfigMap.

Examples:

DB_HOST
DB_NAME
This separates application configuration from the container image.

Simon Deployment
The Simon Flask application runs through its own Kubernetes Deployment.

The application container receives configuration from:

ConfigMap + Secret

The application listens on port 5000.

Simon Service
A ClusterIP Service exposes the Simon application inside the Kubernetes cluster.

The Service provides a stable network endpoint even if the application Pod is recreated.

Simon Ingress
External HTTP access is provided through NGINX Ingress.

The application is exposed using:

simon.local

The Ingress routes requests to the Simon Service on port 5000.

End-to-End Request Flow
The complete application request path is:

Windows Notebook ↓ hosts file ↓ simon.local ↓ RHEL 9 ↓ Docker ↓ Kind control-plane ↓ NGINX Ingress Controller ↓ Simon Ingress ↓ Simon Service ↓ Simon Pod ↓ Flask Application

For database operations:

Flask Application ↓ PostgreSQL Service ↓ PostgreSQL Pod ↓ PVC

Validation Strategy
An important principle used throughout this lab was:

Create one component, validate it, and only then move to the next layer.

For the Simon application, the sequence was approximately:

Create the namespace
Create the PostgreSQL Secret
Create the PostgreSQL PVC
Deploy PostgreSQL
Create the PostgreSQL Service
Validate the database
Create the Simon ConfigMap
Load the Simon container image into Kind
Deploy the Simon application
Validate the application logs
Create the Simon Service
Test the Service through port forwarding
Validate /health
Create the Ingress
Configure local hostname resolution
Access the application from an external notebook
The /health endpoint confirmed communication between the Flask application and PostgreSQL:

{
  "database": "connected",
  "status": "ok"
}
