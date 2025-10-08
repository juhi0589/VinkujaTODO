# VinkujaTODO 

Helm chart for deploying **Vikunja** (open-source task and project management app) on Kubernetes.  
This repo demonstrates a **case study** on deploying Vikunja with **GKE (Google Kubernetes Engine)**, **Helm**, and **Postgres**.


## Live Application

Vikunja is deployed on a GKE regional cluster using Helm.  
Access the live app here:  

👉 http://34.59.253.239:3456

---

## 1) Executive Summary
- Deployed Vikunja (web + API) on a **GKE regional cluster** using Helm.  
- Exposed via **Service type: LoadBalancer (L4, regional)**.  
- Uses **in-cluster Postgres (StatefulSet)**.  
- Storage: **dynamic PVCs**.  
- Observability: **Prometheus/Grafana** enabled; metrics from `/api/v1/metrics`.  

---

## 2) Architecture Overview
- **Container:** `vikunja/vikunja` (serves Web UI + REST API on port 3456).  
- **Database:** Postgres release `mypg-postgresql`, reachable at  
  `mypg-postgresql.keycloak.svc.cluster.local:5432`.  
- **Exposure:** L4 LoadBalancer → public IP.  
- **Storage:**  
  - PVC `vikunja-storage` mounted at `/app/vikunja/files`.  
  - One PVC per Postgres replica (StatefulSet).  

**Why split this way:**  
- Stateless web/API → **Deployment**.  
- Stateful DB → **StatefulSet** (stable identity + one PVC per replica).  

---

## 3) Why Helm
- Parametric, repeatable installs via values.  
- Chart features: service, PVC, ServiceMonitor hooks.  
- Lifecycle safety: `helm upgrade --install` for idempotent deploys/rollbacks.  

---

## 4) Environment (GKE Regional)
- Control plane & nodes across zones (higher availability).  
- Service LoadBalancer = **L4 regional**.  
- StorageClasses: `standard-rwo` (default).  

---

## 5) Deployment — High Level Steps
1. Namespace: `vikunja`.  
2. DB Secret: create `vikunja-db` (keys: `username`, `password`) in namespace.  
3. Values: set `VIKUNJA_DATABASE_*` to Postgres service and use Secret refs.  
4. Expose: Service type: LoadBalancer.  
5. Scale & harden:  
   - Replicas ≥ 2.  
   - Readiness & liveness probes on `/health` (3456).  
   - Resource requests/limits defined.  
   - Spread replicas across zones.  
6. Observability: enable ServiceMonitor; verify metrics in Grafana.  
7. Sanity checks:  
   - Pods Ready.  
   - LB has external IP.  
   - Logs clean.  
   - UI flows work.  


## 6) Network & Exposure
- **Now:** L4 LoadBalancer — minimal moving parts, good for one service.  
- **Later:** L7/TLS via nginx Ingress + cert-manager when host/path routing or HTTPS is required.  


## 7) Database Strategy — Why Self-Hosted Now
1. **Portability:** Same chart/values run on any K8s (dev/on prem/cloud).  
2. **Low latency:** App and DB in the same cluster over internal DNS.  
3. **Cost:** Managed DBs bill continuously; for short term testing/learning, self-hosting is chep on existing cluster.  
4. **Cloud agnostic:** Vendor neutral and portable across environments.  


## 8) Storage
- Mode: Dynamic provisioning via default `standard-rwo`.  
- App PVC: `vikunja-storage` (RWO) at `/app/vikunja/files`.  
- DB PVC: one RWO PVC per StatefulSet pod.  

---

## 9) Observability & Metrics
- **Prometheus:** ServiceMonitor scrapes `/api/v1/metrics` every 30s.  
- **Grafana (running):**  
  ```bash
  kubectl port-forward service/prometheus-grafana 3000:80 -n vikunja

