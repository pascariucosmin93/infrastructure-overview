# Infrastructure Overview

Personal homelab running a self-hosted Kubernetes cluster on Proxmox, managed with GitOps and monitored with a full observability stack.

## Architecture

```mermaid
graph TB
    subgraph Internet
        CF[Cloudflare]
        GH[GitHub]
        GHCR[GHCR\nContainer Registry]
    end

    subgraph Proxmox["Proxmox Hypervisor"]
        subgraph K8S["Kubernetes Cluster — Cilium CNI + BGP"]
            direction TB

            subgraph ControlPlane["Control Plane (m — 192.168.70.20)"]
                API[kube-apiserver]
                ARGO[ArgoCD]
            end

            subgraph Workers["Worker Nodes (w1/w2/w3 — 192.168.70.21-23)"]
                direction LR
                subgraph Apps["Applications"]
                    CV[cv-website]
                    GAZ[gaz]
                    EL[employee-leave]
                    TODO[todo-app]
                    IMMICH[Immich]
                    FORGEJO[Forgejo]
                end

                subgraph Monitoring["Monitoring"]
                    PROM[Prometheus]
                    GRAF[Grafana]
                    ALLOY[Alloy]
                end
            end

            subgraph Networking["Networking"]
                CILIUM[Cilium\nkube-proxy replacement]
                BGP[BGP Control Plane\nASN 65001]
            end
        end

        LOKI[Loki\n10.13.13.30]
        CSI[Proxmox CSI\nStorage]
    end

    subgraph Routers["BGP Routers (ASN 65000)"]
        R1[Router 1\n192.168.70.9]
        R2[Router 2\n192.168.70.10]
    end

    subgraph Alerting
        DISCORD[Discord]
    end

    %% External traffic
    CF -->|Cloudflare Tunnel| CV
    CF -->|Cloudflare Tunnel| GAZ

    %% GitOps flow
    GH -->|push triggers CI| GHCR
    GHCR -->|image pull| K8S
    GH -->|GitOps sync| ARGO
    ARGO -->|deploys| Apps

    %% BGP
    BGP <-->|eBGP peering| R1
    BGP <-->|eBGP peering| R2

    %% Observability
    ALLOY -->|logs| LOKI
    ALLOY -->|scrapes pod logs| Workers
    PROM -->|scrapes metrics| Workers
    PROM -->|remote write| PROM
    LOKI -->|datasource| GRAF
    PROM -->|datasource| GRAF
    GRAF -->|alerts| DISCORD

    %% Storage
    CSI -->|PersistentVolumes| Monitoring
    CSI -->|PersistentVolumes| FORGEJO
    CSI -->|PersistentVolumes| IMMICH
```

## Stack

| Layer | Technology |
|-------|-----------|
| Hypervisor | Proxmox VE |
| OS | Ubuntu 22.04 (provisioned with Ansible) |
| Kubernetes | kubeadm, v1.31 |
| CNI | Cilium (kube-proxy replacement, BGP control plane) |
| GitOps | ArgoCD |
| CI/CD | GitHub Actions → GHCR |
| Ingress | Cloudflare Tunnel |
| Storage | Proxmox CSI Plugin |
| Monitoring | Prometheus + Grafana + Alloy + Loki |
| Alerting | Grafana → Discord webhooks |
| Git hosting | Forgejo (self-hosted) + GitHub |

## Repositories

| Repository | Description |
|------------|-------------|
| [cv-website](https://github.com/pascariucosmin93/cv-website) | Personal CV — static site with Docker + GitHub Actions CI/CD |
| [cv-gitops](https://github.com/pascariucosmin93/cv-gitops) | GitOps manifests for cv-website (ArgoCD + Kustomize) |
| [k8s-sh](https://github.com/pascariucosmin93/k8s-sh) | Shell scripts for bootstrapping the Kubernetes cluster |
| [vm-bootstrap-ansible](https://github.com/pascariucosmin93/vm-bootstrap-ansible) | Ansible roles for VM provisioning and security hardening |
| [k8s-network-policies](https://github.com/pascariucosmin93/k8s-network-policies) | Kubernetes + CiliumNetworkPolicy for all namespaces |
| [monitoring-stack](https://github.com/pascariucosmin93/monitoring-stack) | Helm values and ArgoCD apps for the observability stack |
| [calculatorgaz](https://github.com/pascariucosmin93/calculatorgaz) | Gas price simulator — TypeScript microservices app |

## Network

- **BGP ASN**: 65001 (cluster) peering with 65000 (routers)
- **LoadBalancer pools**: `10.30.10.0/24` (apps), `10.40.10.0/24` (internal tools)
- **External access**: Cloudflare Tunnel (no open ports on the router)
- **Network policies**: default-deny per namespace with explicit allow rules
