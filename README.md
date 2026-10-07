# Homelab

A GitOps-managed, bare-metal Kubernetes homelab. Everything running in the cluster is declared in this repository and reconciled by [Flux CD](https://fluxcd.io/), so Git is the source of truth: if it isn't in the repo, it isn't in the cluster.

This is a living project. I'm building it to learn how infrastructure fits together and, just as importantly, *why* each decision gets made, so the notes below capture the reasoning as well as the setup.

---

## Overview

| Layer | Choice |
|---|---|
| Kubernetes distribution | [k3s](https://k3s.io/) |
| GitOps | Flux CD (with SOPS + age for secrets) |
| Storage | Longhorn (replicated, cluster-local) |
| Databases | CloudNativePG |
| Ingress | Traefik (bundled with k3s) |
| Public access | Cloudflare Tunnel (`cloudflared`, 2 replicas) |
| Monitoring | kube-prometheus-stack (Prometheus + Grafana) |
| Dependency updates | Renovate |
| Networking | UniFi, dedicated VLAN for the homelab |

---

## Hardware

| Role | Machine | RAM |
|---|---|---|
| Control plane | Dell OptiPlex 5000 | 16 GB |
| Worker | Dell OptiPlex 5000 | 32 GB |
| Worker | Dell OptiPlex 5000 | 32 GB |

Each node has a single NVMe drive with a swap size explicitly assigned. I verified RAM and SSD health (memory diagnostics, SMART data) before putting them into service.

**Network and rack:** UniFi Cloud Gateway Max, UniFi 210 switch, 2.5GbE flex switch, UniFi AP7 (PoE), Raspberry Pi (PoE), and a CyberPower 1000W UPS.

---

## Repository Structure

```
.
├── clusters/         # Flux entrypoints and cluster-level configuration
├── infrastructure/   # Platform components the apps depend on
├── apps/             # Application workloads, one folder per app
└── monitoring/       # Observability stack
```

- **`clusters/`** is where Flux bootstraps from. It points at everything else.
- **`infrastructure/`** holds the platform layer: storage, databases, and tooling that applications build on.
- **`apps/`** holds the workloads. Each app gets its own folder, so adding a new service is just adding a new folder.
- **`monitoring/`** keeps observability separate from application workloads.

Some components live in their own namespaces rather than a shared `apps` namespace (for example `forgejo` and `homepage`), which keeps ownership and blast radius clearer.

---

## What's Running

### `apps/`
- **Audiobookshelf**: self-hosted audiobook and podcast server
- **Forgejo**: self-hosted Git, deployed with a Flux OCIRepository and HelmRelease. Exposed via Cloudflare Tunnel so I can reach it away from home, with push-mirroring to GitHub
- **Homepage**: dashboard for the lab, deployed with a Flux HelmRepository and HelmRelease
- **Linkding**: self-hosted bookmark manager, exposed via Cloudflare Tunnel
- **n8n**: workflow automation, backed by a CloudNativePG-managed Postgres database
- **pgAdmin**: web UI for managing and inspecting the Postgres databases running in the cluster

### `infrastructure/`
- **CloudNativePG**: operator that manages the PostgreSQL databases for apps
- **Longhorn**: replicated persistent storage; all stateful workloads use it
- **Renovate**: scans manifests for outdated container images and chart versions

### `monitoring/`
- **kube-prometheus-stack**: Prometheus and Grafana, internal-only and reached through Traefik using an internal hostname. Deliberately *not* routed through the public tunnel

---

## Design Decisions

**k3s without its bundled helm-controller.** Flux ships its own helm-controller, and running both causes conflicts. k3s is installed with `--disable=helm-controller` on the control plane so Flux owns Helm releases.

**Public vs. internal exposure.** Services I need from outside the house (Linkding, Forgejo) go through a Cloudflare Tunnel, so no ports are opened on the router. Anything operational, like Prometheus and Grafana, stays internal.

**Databases through an operator.** CloudNativePG manages PostgreSQL as a declarative, Longhorn-backed resource instead of hand-written StatefulSets, so database setup stays GitOps-native.

**Secrets stay encrypted in Git.** Secrets are encrypted with SOPS and age before being committed. No plaintext credentials live in this repository.

---

## Roadmap

- Offsite Longhorn backups to AWS S3
- Cilium as the CNI

---

## Learning Notes

This repo doubles as a public record of what I'm learning. Along the way I've worked through Kubernetes networking, volumes and mounts, Secrets and ConfigMaps, kubeconfig management, and how Flux's kustomize-controller handles auto-generated versus explicit `kustomization.yaml` files.
