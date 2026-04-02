# Infrastructure Overview

Personal homelab and cloud infrastructure — self-hosted Kubernetes cluster on Proxmox, AWS and Azure environments provisioned with Terraform, managed with GitOps and monitored with a full observability stack.

## Homelab Architecture

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

    CF -->|Cloudflare Tunnel| CV
    CF -->|Cloudflare Tunnel| GAZ

    GH -->|push triggers CI| GHCR
    GHCR -->|image pull| K8S
    GH -->|GitOps sync| ARGO
    ARGO -->|deploys| Apps

    BGP <-->|eBGP peering| R1
    BGP <-->|eBGP peering| R2

    ALLOY -->|logs| LOKI
    ALLOY -->|scrapes pod logs| Workers
    PROM -->|scrapes metrics| Workers
    LOKI -->|datasource| GRAF
    PROM -->|datasource| GRAF
    GRAF -->|alerts| DISCORD

    CSI -->|PersistentVolumes| Monitoring
    CSI -->|PersistentVolumes| FORGEJO
    CSI -->|PersistentVolumes| IMMICH
```

## Cloud Architecture

```mermaid
graph LR
    subgraph AWS["AWS — Terraform"]
        direction TB
        VPC[VPC\nsubnets + routing]
        EKS[EKS Cluster]
        ECR[ECR]
        S3[S3]
        IAM[IAM roles\n+ policies]
        VPC --> EKS
        ECR --> EKS
        IAM --> EKS
    end

    subgraph Azure["Azure — Terraform"]
        direction TB
        VNET[VNet]
        AKS[AKS Cluster]
        ACR[Azure Container\nRegistry]
        VNET --> AKS
        ACR --> AKS
    end

    TF[Terraform] -->|provisions| AWS
    TF -->|provisions| Azure
```

## Stack

### Homelab

| Layer | Technology |
|-------|-----------|
| Hypervisor | Proxmox VE |
| OS provisioning | Ansible |
| Kubernetes | kubeadm v1.31 |
| CNI | Cilium (kube-proxy replacement, BGP control plane) |
| GitOps | ArgoCD |
| CI/CD | GitHub Actions → GHCR |
| Ingress | Cloudflare Tunnel |
| Storage | Proxmox CSI Plugin |
| Monitoring | Prometheus + Grafana + Alloy + Loki |
| Alerting | Grafana → Discord |
| Git hosting | Forgejo (self-hosted) |

### Cloud

| Layer | Technology |
|-------|-----------|
| IaC | Terraform |
| AWS | VPC, EKS, ECR, S3, IAM |
| Azure | VNet, AKS, ACR |

## Repositories

### Homelab

| Repository | Description |
|------------|-------------|
| [cv-website](https://github.com/pascariucosmin93/cv-website) | Personal CV — static site with Docker + GitHub Actions CI/CD |
| [cv-gitops](https://github.com/pascariucosmin93/cv-gitops) | GitOps manifests for cv-website (ArgoCD + Kustomize) |
| [k8s-sh](https://github.com/pascariucosmin93/k8s-sh) | Shell scripts for bootstrapping the Kubernetes cluster |
| [vm-bootstrap-ansible](https://github.com/pascariucosmin93/vm-bootstrap-ansible) | Ansible roles for VM provisioning and security hardening |
| [k8s-network-policies](https://github.com/pascariucosmin93/k8s-network-policies) | Kubernetes + CiliumNetworkPolicy for all namespaces |
| [monitoring-stack](https://github.com/pascariucosmin93/monitoring-stack) | Helm values and ArgoCD apps for the observability stack |
| [calculatorgaz](https://github.com/pascariucosmin93/calculatorgaz) | Gas price simulator — TypeScript microservices |

### Cloud

| Repository | Description |
|------------|-------------|
| [aws-infra](https://github.com/pascariucosmin93/aws-infra) | AWS infrastructure — VPC, EKS, IAM with Terraform |
| [azure-aks-platform](https://github.com/pascariucosmin93/azure-aks-platform) | Azure AKS platform with Terraform |

## Network

- **BGP ASN**: 65001 (cluster) peering with 65000 (routers)
- **LoadBalancer pools**: `10.30.10.0/24` (apps), `10.40.10.0/24` (internal tools)
- **External access**: Cloudflare Tunnel (no open ports on the router)
- **Network policies**: default-deny per namespace with explicit allow rules
