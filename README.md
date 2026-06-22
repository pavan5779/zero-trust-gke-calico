# Zero Trust Networking on GKE using Calico

This repository contains Kubernetes manifests used in the Medium blog:

**Implementing Enterprise Zero-Trust Networking on GKE using Calico**

The goal of this project is to demonstrate how to secure service-to-service communication in a Google Kubernetes Engine (GKE) cluster using Calico Network Policies.

---

## Architecture Overview

The sample application is an e-commerce platform called **ShopEasy**.

It contains five services:

* **Frontend** – Customer-facing website
* **API** – Business logic service
* **Payment** – Payment processing service
* **Database** – Stores customer and order data
* **AI Services** – Product recommendation service

By default, Kubernetes allows pods to communicate freely with each other. This repository shows how to move from open communication to a Zero Trust model.

---

## Zero Trust Goal

The goal is to allow only required communication paths:

```text
Frontend → API
API → Payment
Payment → Database
```

All other unnecessary communication should remain blocked.

Examples of blocked traffic:

```text
Frontend → Database
Frontend → Payment
Frontend → AI Services
AI Services → Database
AI Services → Payment
API → Database
```

---

## Repository Structure

```text
zero-trust-gke-calico/
├── README.md
├── manifests/
│   ├── 01-shopeasy-app.yaml
│   ├── 02-default-deny.yaml
│   ├── 03-allow-dns.yaml
│   ├── 04-allow-frontend-to-api.yaml
│   ├── 05-allow-api-to-payment.yaml
│   └── 06-allow-payment-to-database.yaml
└── screenshots/
    ├── workflow.png
    ├── before-calico.png
    └── after-calico.png
```

---

## Prerequisites

Before applying these manifests, make sure you have:

* A running GKE cluster
* `kubectl` configured for the cluster
* Calico installed on the cluster
* Access to create namespaces, deployments, services, and network policies

Verify cluster access:

```bash
kubectl get nodes
```

Verify Calico:

```bash
kubectl get pods -n calico-system
```

---

## Apply Order

Apply the manifests in this order:

```bash
kubectl apply -f manifests/01-shopeasy-app.yaml
kubectl apply -f manifests/02-default-deny.yaml
kubectl apply -f manifests/03-allow-dns.yaml
kubectl apply -f manifests/04-allow-frontend-to-api.yaml
kubectl apply -f manifests/05-allow-api-to-payment.yaml
kubectl apply -f manifests/06-allow-payment-to-database.yaml
```

---

## Manifest Explanation

### 01-shopeasy-app.yaml

This file creates the sample ShopEasy application.

It includes:

* Five namespaces
* Five deployments
* Five services

Namespaces created:

```text
frontend
api
payment
database
ai-services
```

Application components:

```text
frontend     → network-multitool
api          → nginx
payment      → nginx
database     → mysql:8.0
ai-service   → nginx
```

The `frontend` pod uses `praqma/network-multitool` because it includes tools such as `curl`, `wget`, `ping`, and `nc`, which are useful for testing network connectivity.

Verify resources:

```bash
kubectl get ns
kubectl get pods -A
kubectl get svc -A
kubectl get deployments -A
```

---

### 02-default-deny.yaml

This file applies a Default Deny policy to each application namespace.

Namespaces protected:

```text
frontend
api
payment
database
ai-services
```

The policy blocks:

* All ingress traffic
* All egress traffic

This is the foundation of Zero Trust networking.

After applying this file, all pod-to-pod communication is blocked unless explicitly allowed.

Verify policies:

```bash
kubectl get networkpolicy -A
```

Expected behavior:

```text
Frontend → API        Blocked
Frontend → Database   Blocked
API → Payment         Blocked
Payment → Database    Blocked
```

---

### 03-allow-dns.yaml

After applying Default Deny, DNS lookups may also be blocked.

This file allows egress traffic from application namespaces to Kubernetes DNS running in the `kube-system` namespace.

DNS is required because service names such as the following must resolve correctly:

```text
api-service.api.svc.cluster.local
payment-service.payment.svc.cluster.local
database-service.database.svc.cluster.local
```

Without DNS access, service-to-service testing using service names may fail even if the network policy is correct.

This file allows UDP traffic on port `53` to `kube-dns`.

---

### 04-allow-frontend-to-api.yaml

This file allows only the required traffic from the Frontend service to the API service.

It creates two policy rules:

* Allows egress from `frontend` namespace to `api` namespace
* Allows ingress into the API pod from the Frontend pod

Allowed path:

```text
Frontend → API
```

Test from the frontend pod:

```bash
kubectl exec -it <frontend-pod-name> -n frontend -- sh

curl api-service.api.svc.cluster.local
```

Expected output:

```text
Welcome to nginx!
```

Database access should still be blocked:

```bash
nc -zv database-service.database.svc.cluster.local 3306
```

Expected result:

```text
Connection timed out
```

---

### 05-allow-api-to-payment.yaml

This file allows traffic from API to Payment.

It creates:

* Egress policy from API to Payment
* Ingress policy into Payment from API

Allowed path:

```text
API → Payment
```

This simulates the API service calling the Payment service during checkout.

---

### 06-allow-payment-to-database.yaml

This file allows traffic from Payment to Database.

It creates:

* Egress policy from Payment to Database
* Ingress policy into Database from Payment

Allowed path:

```text
Payment → Database
```

This simulates the Payment service writing transaction or order data into the database.

---

## Testing Connectivity

Get the frontend pod name:

```bash
kubectl get pods -n frontend
```

Access the frontend pod:

```bash
kubectl exec -it <frontend-pod-name> -n frontend -- sh
```

Test allowed traffic:

```bash
curl api-service.api.svc.cluster.local
```

Expected result:

```text
Welcome to nginx!
```

Test blocked traffic:

```bash
nc -zv database-service.database.svc.cluster.local 3306
```

Expected result:

```text
Connection timed out
```

---

## Final Validation

Allowed communication:

```text
Frontend → API
API → Payment
Payment → Database
```

Blocked communication:

```text
Frontend → Database
Frontend → Payment
Frontend → AI Services
AI Services → Database
AI Services → Payment
API → Database
```

This confirms that Zero Trust networking is working correctly.

---

## Cleanup

To remove the demo resources:

```bash
kubectl delete -f manifests/06-allow-payment-to-database.yaml
kubectl delete -f manifests/05-allow-api-to-payment.yaml
kubectl delete -f manifests/04-allow-frontend-to-api.yaml
kubectl delete -f manifests/03-allow-dns.yaml
kubectl delete -f manifests/02-default-deny.yaml
kubectl delete -f manifests/01-shopeasy-app.yaml
```

---

## Security Notes

This repository is for demo and learning purposes.

For production environments, consider adding:

* Private GKE clusters
* Workload Identity
* Google Secret Manager
* Binary Authorization
* Vulnerability scanning
* Audit logging
* Policy-as-code validation
* GitOps-based policy management

---

## Summary

This repository demonstrates how to use Calico Network Policies on GKE to implement a Zero Trust security model.

The key idea is simple:

```text
Deny everything by default.
Allow only the traffic that the application actually needs.
```

This reduces the attack surface, limits lateral movement, and improves Kubernetes workload security.
