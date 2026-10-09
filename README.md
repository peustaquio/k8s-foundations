# k8s-foundations

A hands-on Kubernetes learning lab built with **RHEL 9, Docker and Kind**.

The main goal of this repository is not simply to deploy applications, but to understand how Kubernetes components work together, validate each layer independently, and practice troubleshooting in a real lab environment.

---

## Overview

This lab was built incrementally in two phases:

1. **Kubernetes Foundations** using NGINX.
2. **Simon Game on Kubernetes**, migrating an existing Flask + PostgreSQL application from Docker Compose to Kubernetes.

The focus throughout the lab was simple:

> Build one component, validate it, understand it, and only then move to the next layer.

---

# Phase 1 - Kubernetes Foundations

The first phase focused on understanding the main Kubernetes building blocks.

## Topics Practiced

- Kind multi-node cluster
- Control plane and worker nodes
- Namespaces
- Deployments
- ReplicaSets
- Pods
- Services
- ConfigMaps
- Secrets
- PersistentVolumeClaims (PVC)
- NGINX Ingress Controller
- Host-based routing
- Kubernetes internal DNS
- Container networking
- Troubleshooting

---

## Basic Architecture

The initial NGINX lab followed this request path:

```text
Windows Notebook
        |
        v
Local DNS resolution (hosts)
        |
        v
RHEL 9
        |
        v
Docker
        |
        v
Kind
        |
        v
NGINX Ingress Controller
        |
        v
Ingress
        |
        v
Service
        |
        v
NGINX Pod
```

---

## Ingress Troubleshooting

One of the most valuable exercises in this lab was troubleshooting external access to the NGINX application.

The NGINX application and Kubernetes resources were healthy, but the application was initially inaccessible from outside the cluster.

The investigation included validating:

- Pods
- Deployments
- Services
- Service endpoints
- Ingress rules
- Ingress Controller logs and configuration
- Docker port mappings
- Kubernetes node placement

The problem was eventually isolated to the topology.

Ports **80** and **443** were mapped through the Kind control-plane node, while the NGINX Ingress Controller was running on another worker node.

After moving the Ingress Controller to the appropriate node, the behavior changed from:

```text
Connection refused
        |
        v
HTTP 404
        |
        v
Welcome to nginx!
```

Each result provided additional information about how far the request was traveling.

The HTTP `404` was especially useful because it demonstrated that traffic was finally reaching the Ingress Controller.

The final step was using the hostname configured in the Ingress rule:

```text
nginx-dev.local
```

The Windows `hosts` file provided local name resolution:

```text
nginx-dev.local
        |
        v
RHEL 9 IP
        |
        v
Ingress
        |
        v
Service
        |
        v
NGINX Pod
```

### Main troubleshooting lesson

> Understand the complete request path first, then validate and eliminate one layer at a time.

---

# Phase 2 - Simon Game on Kubernetes

After validating the Kubernetes fundamentals with NGINX, the next objective was to deploy a real application.

The **Simon Game** is a memory game built with:

- Python
- Flask
- PostgreSQL
- Docker

The application includes a leaderboard backed by PostgreSQL.

Originally, the complete environment was running with Docker Compose.

---

## Original Docker Compose Architecture

```text
Docker Compose
|
+-- Simon / Flask
|
+-- PostgreSQL
    |
    +-- Persistent Docker Volume
```

Instead of automatically converting the Compose environment, the Kubernetes version was built manually, component by component.

This made it possible to understand what each Docker Compose feature becomes in Kubernetes.

---

# Simon Kubernetes Architecture

The final application path is:

```text
Windows Notebook
        |
        v
simon.local
        |
        v
RHEL 9
        |
        v
Docker
        |
        v
Kind
        |
        v
NGINX Ingress Controller
        |
        v
Simon Ingress
        |
        v
Simon Service :5000
        |
        v
Simon Pod
        |
        v
Flask Application
        |
        v
PostgreSQL Service :5432
        |
        v
PostgreSQL Pod
        |
        v
PersistentVolumeClaim
```

---

## Namespace

A dedicated namespace was created:

```text
simon
```

The namespace keeps the Simon resources logically isolated from the other workloads in the lab.

---

## PostgreSQL Secret

Database credentials are provided through a Kubernetes Secret.

The application receives sensitive values such as:

```text
DB_USER
DB_PASSWORD
```

This separates credentials from normal application configuration.

---

## PostgreSQL Persistent Storage

PostgreSQL uses a **PersistentVolumeClaim (PVC)**.

This separates the lifetime of the database data from the lifetime of the PostgreSQL Pod.

Conceptually:

```text
Pod lifecycle
      !=
Data lifecycle
```

A Pod can be replaced while its persistent data remains available through the volume.

---

## PostgreSQL Deployment

PostgreSQL runs as a Kubernetes Deployment using:

```text
postgres:16
```

The PostgreSQL container mounts the persistent storage provided by the PVC.

---

## PostgreSQL Service

A `ClusterIP` Service exposes PostgreSQL internally.

Instead of using the PostgreSQL Pod IP address, the Simon application uses:

```text
DB_HOST=postgres
```

Kubernetes internal DNS resolves `postgres` to the PostgreSQL Service.

Therefore:

```text
Simon Pod
    |
    | DB_HOST=postgres
    v
PostgreSQL Service
    |
    v
PostgreSQL Pod
```

The application does not need to know the PostgreSQL Pod IP.

---

## Simon ConfigMap

Non-sensitive configuration is stored in a ConfigMap.

Examples include:

```text
DB_HOST
DB_NAME
```

This separates application configuration from the container image.

---

## Simon Deployment

The Simon Flask application runs through its own Kubernetes Deployment.

The application receives configuration from two sources:

```text
ConfigMap
   +
Secret
   |
   v
Simon Deployment
   |
   v
Simon Pod
```

The Flask application listens on:

```text
5000
```

The application image was built locally and loaded into the Kind cluster.

---

## Simon Service

A `ClusterIP` Service exposes the Simon application inside Kubernetes.

The Service listens on port `5000` and routes traffic to the Simon Pod.

```text
Simon Service :5000
        |
        v
Simon Pod :5000
```

This provides a stable endpoint independently of the Pod IP.

---

## Internal Validation

Before exposing the application through Ingress, the Simon Service was validated using port forwarding:

```bash
kubectl port-forward svc/simon 5001:5000 -n simon
```

The application health endpoint was then tested:

```bash
curl http://localhost:5001/health
```

Response:

```json
{
  "database": "connected",
  "status": "ok"
}
```

This confirmed that the complete internal path was functioning:

```text
Service
   |
   v
Simon Pod
   |
   v
Flask
   |
   v
PostgreSQL Service
   |
   v
PostgreSQL Pod
```

---

## Simon Ingress

After validating the internal application path, an Ingress was created.

The application hostname is:

```text
simon.local
```

The Ingress routes HTTP traffic to:

```text
simon.local
     |
     v
Simon Service :5000
     |
     v
Simon Pod :5000
```

Local hostname resolution was configured on the Windows notebook using the `hosts` file.

The application was then successfully accessed through:

```text
http://simon.local
```

At this point, the application and its PostgreSQL-backed leaderboard were accessible from the external notebook through Kubernetes Ingress.

---

# Docker Compose to Kubernetes

One of the objectives of this exercise was understanding how concepts from Docker Compose map into Kubernetes.

### Docker Compose

```text
services.postgres
services.simon
environment variables
Docker volume
ports
```

### Kubernetes

```text
Deployments
Services
ConfigMaps
Secrets
PersistentVolumeClaim
Ingress
```

The application architecture remained conceptually similar, while Kubernetes introduced orchestration and abstraction around the containers.

---

# Validation Strategy

Every layer was created and validated before moving to the next one.

The approximate deployment sequence was:

```text
1. Namespace
        |
        v
2. PostgreSQL Secret
        |
        v
3. PostgreSQL PVC
        |
        v
4. PostgreSQL Deployment
        |
        v
5. PostgreSQL Service
        |
        v
6. Database validation
        |
        v
7. Simon ConfigMap
        |
        v
8. Simon image loaded into Kind
        |
        v
9. Simon Deployment
        |
        v
10. Simon logs validation
        |
        v
11. Simon Service
        |
        v
12. Port-forward validation
        |
        v
13. /health validation
        |
        v
14. Simon Ingress
        |
        v
15. Local hostname resolution
        |
        v
16. External browser access
```

The philosophy was:

> Create → Validate → Understand → Continue

---

# Troubleshooting Approach

One of the main lessons from this project was learning not to treat every error as simply a "Kubernetes problem."

Instead, first visualize the complete architecture:

```text
Client
  |
  v
DNS
  |
  v
Host
  |
  v
Container Runtime
  |
  v
Kubernetes Node
  |
  v
Ingress Controller
  |
  v
Ingress
  |
  v
Service
  |
  v
Pod
  |
  v
Application
  |
  v
Database Service
  |
  v
Database Pod
  |
  v
Storage
```

Then determine which layers have already been validated.

Useful commands practiced during the lab:

```bash
kubectl get pods -A
kubectl get pods -o wide
kubectl get deployments -A
kubectl get svc -A
kubectl get ingress -A
kubectl get pvc -A

kubectl describe pod <pod>
kubectl describe svc <service>
kubectl describe ingress <ingress>

kubectl logs <pod>
kubectl logs <pod> --previous
```

Additional tools used for validation:

```bash
curl
ping
docker ps
docker images
```

The important part is not memorizing commands.

The important part is answering:

1. What is the expected request path?
2. Which layers are confirmed working?
3. Which layer has not yet been validated?
4. At which layer does the observed behavior change?

---

# Key Lessons

Some of the main lessons from this lab:

- Pods are disposable.
- Application state should not depend on the Pod lifecycle.
- Services provide stable access to dynamic Pods.
- Kubernetes DNS allows applications to use Service names instead of Pod IP addresses.
- ConfigMaps separate normal configuration from application images.
- Secrets separate sensitive configuration from normal configuration.
- PVCs provide persistent storage independently of Pods.
- Ingress provides HTTP routing into applications.
- Host-based routing depends on the requested hostname.
- A healthy Pod does not necessarily mean the complete application path is healthy.
- Different errors can represent different stages of the request path.
- Troubleshooting becomes easier when the complete architecture is understood first.
- Validate each component before adding another layer.

---

# Next Steps

Future improvements for this lab include:

- [ ] Add readiness probes
- [ ] Add liveness probes
- [ ] Configure CPU requests and limits
- [ ] Configure memory requests and limits
- [ ] Intentionally test Pod self-healing
- [ ] Test PostgreSQL persistence after Pod recreation
- [ ] Experiment with multiple application replicas
- [ ] Test application scaling
- [ ] Improve secret management
- [ ] Replace the Flask development server with a production WSGI server
- [ ] Explore NetworkPolicies
- [ ] Explore Helm
- [ ] Deploy the architecture to a managed Kubernetes environment

---

# Why This Repository Exists

This repository represents my practical Kubernetes learning journey.

The objective is not to build the most complex cluster possible.

The objective is to understand the fundamentals deeply enough that concepts such as:

```text
Deployment
Service
Pod
ConfigMap
Secret
PVC
Ingress
DNS
Networking
```

become natural parts of everyday troubleshooting.

NGINX was used to learn the individual Kubernetes components.

The Simon Game was used to connect those components into a complete application.

The most important lesson so far is:

> Kubernetes becomes much easier to troubleshoot once the complete path between the user, the application, and its dependencies can be visualized.
