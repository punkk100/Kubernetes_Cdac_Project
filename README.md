Kubernetes Infrastructure: Automated Setup, Monitoring & Security
This repository contains the full configuration for a production-ready Kubernetes cluster deployment on Ubuntu. The project focuses on three core pillars: Automation with Ansible, Stateful Persistence with TrueNAS, and Risk-Based Security with kube-scan.

 Project Architecture
Orchestration Layer: Cluster initialized using kubeadm with nodes provisioned via Ansible playbooks for consistent environment setup.

Storage Layer: External TrueNAS server integration via NFS, providing persistent storage for stateful workloads like Nginx and Grafana.

Observability Stack: Real-time metrics collection using Prometheus and visualization via Grafana dashboards.

Security Layer: Automated configuration auditing using kube-scan to identify and remediate deployment risks.

 Tech Stack
OS: Ubuntu Linux (16.04)

Automation: Ansible

Orchestration: Kubernetes (v1.2x +)

Storage: TrueNAS (NFS Protocol)

Monitoring: Prometheus, Grafana, Alertmanager

Security: kube-scan (KCCSS framework)

 Key Implementation Details
 
Stateful Persistence
Unlike ephemeral clusters, this setup uses a dedicated TrueNAS backend (IP: 192.168.30.150). By configuring Persistent Volumes (PV) and Claims (PVC), data survives pod restarts and cluster upgrades.

Risk-Based Security
Integrated kube-scan to analyze the security posture of running workloads. The tool assigns a risk score (0-10) based on configuration vulnerabilities, allowing for prioritized remediation of security gaps.

Automated Alerting
Configured Alertmanager with Gmail SMTP integration to ensure immediate notification for critical system events, such as node failures or high resource utilization.

 Repository Contents
storage.yaml: NFS-based PV and PVC definitions.

prometheus.yaml: RBAC, ConfigMaps, and Deployment for metrics collection.

grafana.yaml: Dashboard deployment linked to persistent storage.

alertmanager.yaml: SMTP and notification routing logic.

security-kube-scan.yaml: Security auditing engine deployment
