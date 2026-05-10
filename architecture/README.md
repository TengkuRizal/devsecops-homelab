# DevSecOps Homelab — Architecture

Enterprise-style DevSecOps homelab with segmented SOC, target, attacker and VPN zones, running GitLab CI/CD, Kubernetes, Wazuh SIEM, Security Onion monitoring, Shuffle SOAR automation, and cloud/IaC security practice.

![Platform](https://img.shields.io/badge/Platform-Proxmox-E57000)
![SIEM](https://img.shields.io/badge/SIEM-Wazuh-005571)
![K8s](https://img.shields.io/badge/Orchestration-Kubernetes-326CE5)
![CI/CD](https://img.shields.io/badge/CI/CD-GitLab-FC6D26)
![SOAR](https://img.shields.io/badge/SOAR-Shuffle-blueviolet)

---

## Overview

This homelab simulates a practical enterprise DevSecOps environment demonstrating an end-to-end security workflow:

**Code → Scan → Build → Deploy → Monitor → Detect → Investigate → Respond**

Built on 4 Mini PC i7 nodes running Proxmox, segmented into four security zones: SOC, Target, Attacker, and VPN.

---

## Physical Infrastructure

| Device | Role | IP |
|---|---|---|
| Mini PC 1 | Proxmox — SOC Zone | 10.10.1.100 |
| Mini PC 2 | Proxmox — Target Zone | 10.10.2.100 |
| Mini PC 3 | Proxmox — Attacker Zone | 10.10.3.100 |
| Mini PC 4 | Proxmox — Security Onion | 10.10.1.151 |
| Mikrotik | WAN Gateway + WireGuard VPN | 192.168.0.1 |
| pfSense | Internal Firewall + inter-VLAN routing | 192.168.0.100 |
| Cisco 2960 | Managed Switch + SPAN Port | Physical |

---

## Network Topology

<pre>
Internet
    │
    ▼
┌──────────────────────────────────────┐
│             Mikrotik                 │
│         192.168.0.1                  │
│      WAN Gateway + WireGuard VPN     │
│                                      │
│  Static routes:                      │
│  ├── 10.10.1.0/24  → pfSense         │
│  ├── 10.10.2.0/24  → pfSense         │
│  ├── 10.10.3.0/24  → pfSense         │
│  └── 10.10.10.0/24 → WireGuard VPN   │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│              pfSense                 │
│          192.168.0.100               │
│   Internal Firewall + inter-VLAN     │
│   DHCP + DNS + ACLs per zone         │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│           Cisco 2960                 │
│         Managed Switch               │
│  SPAN Port ──────────────────────────┼──► Security Onion
└───────┬──────────┬────────┬──────────┘    (passive monitoring)
        │          │        │
        ▼          ▼        ▼
   VLAN 10     VLAN 20  VLAN 30
   SOC          Target  Attacker
   10.10.1.x   10.10.2.x  10.10.3.x
</pre>

---

## Security Zone Segmentation

<pre>
┌─────────────────────────────────────────────────────────────┐
│  VLAN 10 — SOC Zone (10.10.1.0/24)                          │
│                                                             │
│  10.10.1.21    GitLab Runner                                │
│  10.10.1.30    Wazuh SIEM (Manager + Indexer + Dashboard)   │
│  10.10.1.70    Kubernetes Master (k8sMaster)                │
│  10.10.1.100   Proxmox Host 1                               │
│  10.10.1.101   GitLab CE                                    │
│  10.10.1.151   Security Onion                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  VLAN 20 — Target Zone (10.10.2.0/24)                       │
│                                                              │
│  10.10.2.30    Windows Server DC01 — domain lab.local       │
│  10.10.2.x     Windows Client                               │
│  10.10.2.x     DVWA (Damn Vulnerable Web App)               │
│  10.10.2.100   Proxmox Host 2                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  VLAN 30 — Attacker Zone (10.10.3.0/24)                     │
│                                                             │
│  10.10.3.20    Kali Linux                                   │
│  10.10.3.100   Proxmox Host 3                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  VPN Zone (10.10.10.0/24)                                   │
│                                                             │
│  WireGuard on Mikrotik — secure remote access to all zones  │
└─────────────────────────────────────────────────────────────┘
</pre>

---

## Full Stack

### CI/CD Pipeline — GitLab (10.10.1.101)

4-stage security pipeline on every commit:

<pre>
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Stage 1  │    │ Stage 2  │    │ Stage 3  │    │ Stage 4  │
│ Gitleaks │───►│ Semgrep  │───►│  Trivy   │───►│  Deploy  │
│ Secrets  │    │  SAST    │    │Container │    │ kubectl  │
│ Scanning │    │ Scanning │    │ Scanning │    │          │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
</pre>

### Kubernetes Cluster — k8sMaster (10.10.1.70)

- 3-node cluster deployed with kubeadm
- Calico CNI, local-path-provisioner, metrics-server
- kube-prometheus-stack (Prometheus + Grafana)
- GitLab Kubernetes Agent for GitOps deployment
- Wazuh agents as DaemonSet
- Falco runtime security monitoring

### SIEM — Wazuh (10.10.1.30)

- Wazuh Manager + Indexer + Dashboard
- Active agents: Linux nodes, Windows DC01, Windows client
- Custom detection rules mapped to MITRE ATT&CK
- Active response for automated threat blocking
- Webhook integration with Shuffle SOAR

### Network Monitoring — Security Onion (10.10.1.151)

- SPAN port on Cisco 2960 mirrors all network traffic
- Zeek for protocol analysis and connection logs
- Suricata IDS for signature-based detection
- Full packet capture for forensic analysis

### SOAR — Shuffle

- Wazuh webhook triggers automated workflows
- Python-based alert triage script (wazuh_triage.py)
- Hourly cron job for alert correlation and enrichment
- Automated response: email, GitLab issue, IP block

---

## Security Data Flow

<pre>
Attack Simulation (Kali → Target)
            │
            ▼
┌───────────────────────┐
│    Detection Layer    │
│  Wazuh + Zeek +       │
│  Suricata + Falco     │
└──────────┬────────────┘
           │ Alert generated
           ▼
┌───────────────────────┐
│     Wazuh SIEM        │
│  Correlation engine   │
│  MITRE ATT&CK mapping │
└──────────┬────────────┘
           │ Webhook trigger
           ▼
┌───────────────────────┐
│    Shuffle SOAR       │
│  wazuh_triage.py      │
│  Severity assessment  │
└──────────┬────────────┘
           │ Automated response
           ▼
┌───────────────────────┐
│   Active Response     │
│  Block IP             │
│  GitLab issue created │
│  Email notification   │
└───────────────────────┘
</pre>

---

## Attack Simulation

| Component | Detail |
|---|---|
| Attacker | Kali Linux (10.10.3.20) |
| Target — Web | DVWA (Damn Vulnerable Web App) |
| Target — AD | Windows Server DC01, domain lab.local |
| Monitoring | Wazuh + Security Onion via SPAN |

MITRE ATT&CK techniques simulated:
- T1110 — Brute Force
- T1059 — Command and Scripting Interpreter
- T1078 — Valid Accounts
- T1046 — Network Service Discovery
- T1190 — Exploit Public-Facing Application (DVWA)

---

## Cloud & IaC — AWS Terraform

- VPC with multi-AZ public/private subnets
- Least privilege IAM with ARN-scoped policies
- S3 with encryption, versioning, lifecycle management
- VPC Flow Logs → CloudWatch
- Checkov IaC scan: **61 passed, 0 failed**

See: [terraform-aws-devsecops](https://github.com/TengkuRizal/terraform-aws-devsecops)

---

## Remote Access

WireGuard VPN on Mikrotik (192.168.0.1):
- VPN subnet: 10.10.10.0/24
- Routes to all lab zones (10.10.1-3.0/24)
- Secure remote management from anywhere

---

## Interview One-Liner

> I built an enterprise-style DevSecOps homelab with segmented SOC, target, attacker and VPN zones. I run GitLab CI/CD with 4 security gates, deploy to Kubernetes, monitor with Wazuh SIEM and Security Onion, and automate incident response with Shuffle SOAR.

---

## Author

**Tengku Rizal** — DevSecOps Engineer
Building: GitLab CI/CD · Kubernetes · Wazuh SIEM · Terraform · Security Automation
Location: Kuala Lumpur, Malaysia
