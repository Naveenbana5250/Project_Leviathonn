# Chaos & Honeytokens — Defense Team 

> **Scope**: Deploy Chaos Mesh to test platform resilience and strategically plant honeytokens for proactive threat detection. This README documents **what’s in this repo** and **how to run/verify** it — only authoritative paths are referenced, no large file dumps.  



## System Architecture Overview

- Chaos Mesh orchestrates controlled failures (pod kill, network latency) in the `resilience-lab` namespace.  
- Canary/honeytoken application deployed in `security-canary` namespace to detect unauthorized access.  
- Alerts are forwarded to ELK stack for centralized logging and monitoring.  

> This Phase 2 setup builds on Phase 1 infrastructure and security controls (Kyverno policies, signed images) to validate resilience and detection capabilities under stress.


## Outcomes

**Chaos Experiments**  

* Successfully deployed `leviathan-app` with 3 replicas.  
* Executed Chaos Mesh experiments including pod terminations and network latency injections.  
* Verified application self-healing and continued availability during disruptions.  
* Collected experiment events for resilience evaluation.  

**Honeytokens / Detection**  

* Deployed a Flask-based canary application with hardened security context.  
* Honeytokens embedded in ConfigMap and pod environment variables.  
* Alerts generated on honeytoken access and forwarded to ELK stack.  
* Verified alerting and logging mechanism works as expected.  



## Repo Layout (authoritative paths)
### Chaos Engineering

* **Phase2/cluster/chaos/**
    * **apps/** — Contains the `resilience-lab` application used for chaos testing.  
    * **chaos-mesh/**  
        * `install.yaml` — Chaos Mesh installation manifest.  
        * `values.yaml` — Helm configuration for Chaos Mesh.  
        * `experiments/` — Contains all chaos experiment manifests (e.g., pod-kill-experiment.yaml, network-latency-experiment.yaml).  

### Deception Technology (Honeytokens & Honeypot)

* **Phase2/cluster/Honeytoken-detection/**  
    * `canary-elk.yaml` — Deployment, Service, and ConfigMap for the honeypot application.  
    * **canary-app/**  
        * `requirements.txt` — Python dependencies for the canary app.  

## Prerequisites

* Kubernetes cluster accessible from CI or local `kubectl`.  
* Helm v3 (for Chaos Mesh deployment).  
* ELK stack accessible for receiving honeytoken alerts.  
* Docker Hub or other registry for canary image (already built).  


## How to Run / Verify
1. Deploy Resilience Lab Applications
 ```bash
kubectl apply -f Phase2/cluster/apps/resilience-lab/ns.yaml
kubectl apply -f Phase2/cluster/apps/resilience-lab/deployment.yaml
kubectl apply -f Phase2/cluster/apps/resilience-lab/service.yaml
```
* Confirm pods are Running and service is accessible.

2. Deploy Chaos Mesh
 ```bash
kubectl apply -f Phase2/cluster/chaos-mesh/install.yaml
kubectl get po -n chaos-testing
```
* Deploy chaos experiments (pod kill, network latency) from experiments/.
* Validate that pods recover automatically and metrics/events are recorded.

3. Deploy Canary/Honeytokens
```bash
kubectl apply -f Phase2/canary-elk.yaml
kubectl get deploy,svc -n security-canary
```
* Confirm honeytoken pod is running.
* Test alert mechanism by sending HTTP requests to the canary endpoint.
* Verify alerts appear in ELK stack logs.

## Evidence Checklist

* kubectl get deploy,po -n resilience-lab shows all replicas healthy.

* Chaos experiments executed with pod terminations and network perturbations.

* kubectl get events -n resilience-lab confirms Chaos Mesh events triggered.

* Honeytoken canary pod running in security-canary namespace.

* ELK stack logs show honeytoken access alerts.



