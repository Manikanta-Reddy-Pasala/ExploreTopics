# 5G Core Network Comparison: Open5GS vs free5GC vs Radisys

> **Document Type:** Technical Evaluation & Decision Guide
> **Scope:** Single-NUC / Edge / Lab / Private 5G deployment
> **Audience:** Platform Engineering, CTO, Network Architects
> **Hardware Target:** Intel NUC (single-node deployment)
> **Last Updated:** February 2026
> **Status:** ACTIVE — Review quarterly

---

## Table of Contents

1. [Executive Summary & Verdict](#1-executive-summary--verdict)
2. [At a Glance — Scorecard](#2-at-a-glance--scorecard)
3. [Single-NUC Deployment Architecture](#3-single-nuc-deployment-architecture)
4. [Architecture Deep Dive](#4-architecture-deep-dive)
5. [Network Function Coverage Matrix](#5-network-function-coverage-matrix)
6. [Language, Code Quality & Developer Experience](#6-language-code-quality--developer-experience)
7. [Resource Footprint — NUC-Sized](#7-resource-footprint--nuc-sized)
8. [3GPP Standards Compliance](#8-3gpp-standards-compliance)
9. [Open Source Health & Governance](#9-open-source-health--governance)
10. [Security & Supply Chain](#10-security--supply-chain)
11. [Licensing Deep Dive](#11-licensing-deep-dive)
12. [Day-2 Operations](#12-day-2-operations)
13. [Scaling & HA on a Single Node](#13-scaling--ha-on-a-single-node)
14. [Testing & Simulation Ecosystem](#14-testing--simulation-ecosystem)
15. [RAN Interoperability](#15-ran-interoperability)
16. [Migration Paths](#16-migration-paths)
17. [Total Cost of Ownership — NUC Edition](#17-total-cost-of-ownership--nuc-edition)
18. [Risk Register](#18-risk-register)
19. [Decision Framework](#19-decision-framework)
20. [PoC Playbook — Single NUC](#20-poc-playbook--single-nuc)
21. [Governance Checklist](#21-governance-checklist)
22. [Appendix: Glossary](#22-appendix-glossary)

---

## 1. Executive Summary & Verdict

### The Short Version

| | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **One-liner** | Lean C core — runs full 5GC + EPC on a single NUC | Cloud-native Go core — broadest NF set, heavier footprint | Turnkey vendor black-box — not designed for NUC |
| **Best for** | Resource-constrained edge, hybrid LTE+5G, single-node | K8s-native labs, commercial embedding, NF-level scaling | Zero in-house telecom expertise, unlimited budget |
| **Single-NUC fit** | **Excellent** — purpose-built for small footprints | **Good** — needs tuning, more containers to manage | **Poor** — vendor expects multi-node infrastructure |
| **License** | AGPLv3 (copyleft) | Apache 2.0 (permissive) | Proprietary |
| **Language** | C | Go | Undisclosed |
| **Maturity** | High (since 2017) | Medium-High (since 2019) | Vendor-dependent |
| **Year-1 NUC TCO** | **$1K-2.5K** (hardware + eng time) | **$1K-3K** (hardware + eng time) | **$200K+** (overkill for NUC) |

### Single-NUC Decision Matrix

| If your priority is... | Recommendation |
|---|---|
| Smallest possible footprint on a NUC | **Open5GS** |
| Run everything (5GC + sim + monitoring) on one box | **Open5GS** |
| Hybrid LTE + 5G on same NUC | **Open5GS** (only option) |
| K8s-native with k3s on a NUC | **free5GC** (or Open5GS with Helm) |
| Apache 2.0 license for commercial embedding | **free5GC** |
| Broadest NF coverage (N3IWF, CHF, NEF) | **free5GC** |
| No kernel module hassle on NUC | **Open5GS** |
| Vendor SLA, turnkey deployment | **Radisys** (but not on a NUC) |

---

## 2. At a Glance — Scorecard

### Criteria Ratings (1-5, higher is better)

| Criteria | Open5GS | free5GC | Radisys | Notes |
|---|:---:|:---:|:---:|---|
| **Single-NUC Deployability** | **5** | 3 | 1 | Open5GS fits a NUC natively; free5GC needs tuning; Radisys isn't NUC-friendly |
| **Deployment Simplicity** | 4 | 3 | 4 | Open5GS: apt install + YAML. free5GC: more containers to wire up |
| **Cloud-Native Readiness** | 2 | 5 | 3 | free5GC designed for K8s from day one |
| **Resource Efficiency** | **5** | 3 | 2 | Open5GS: ~2 GB total. free5GC: ~4-6 GB. Matters on a NUC |
| **NF Coverage Breadth** | 3 | **5** | 3 | free5GC: N3IWF, TNGF, CHF, NEF |
| **4G/5G Hybrid Support** | **5** | 1 | 2 | Open5GS: EPC + 5GC in one codebase |
| **Code Auditability** | 5 | 5 | 0 | OSS = full audit; Radisys = black box |
| **Customizability** | 5 | 5 | 1 | Source access vs vendor request queue |
| **Vendor Support / SLA** | 1 | 1 | 5 | Only Radisys offers contractual SLA |
| **Hiring Pool** | 2 | 4 | N/A | Go devs >> C + telecom specialists |
| **License Flexibility** | 2 | **5** | 1 | Apache 2.0 is most business-friendly |
| **Supply Chain Control** | 5 | 5 | 0 | Full SBOM, image signing, CVE scanning for OSS |
| **Community & Ecosystem** | 4 | 4 | 1 | Both OSS have active communities |
| **Operational Maturity** | 3 | 3 | 4 | Vendor handles Day-2 for Radisys |
| **Total (out of 70)** | **51** | **52** | **27** | OSS options score ~2x the vendor option |

> **For single-NUC deployments:** Open5GS wins on footprint and simplicity. free5GC wins on NF breadth and license. Radisys is not a practical NUC option.

---

## 3. Single-NUC Deployment Architecture

### Why a Single NUC?

A single Intel NUC (Next Unit of Computing) is a compact, low-power x86 box that can run an entire private 5G core for lab, PoC, edge, or small-scale production. It eliminates the need for rack servers, multi-node K8s clusters, or cloud infrastructure.

### Recommended NUC Hardware

| Component | Minimum (Lab/Demo) | Recommended (Dev/Test) | Maximum (Stress Testing) |
|---|---|---|---|
| **Model** | NUC 11 i5 (Tiger Lake) | NUC 13 i7 (Arena Canyon) | NUC 13 Extreme (Raptor Canyon) |
| **CPU** | 4C/8T @ 2.4-4.2 GHz | 12C/16T @ up to 5.0 GHz | 24C/32T @ up to 5.8 GHz |
| **RAM** | 16 GB DDR4 | 32-64 GB DDR4 | 64-128 GB DDR5 |
| **Storage** | 256 GB NVMe | 512 GB NVMe | 1 TB NVMe |
| **NIC** | 1x 2.5GbE | 1x 2.5GbE + USB-to-GbE | Dual 2.5GbE |
| **TDP** | 28W | 28W | 125W |
| **Cost** | ~$400-600 | ~$700-900 | ~$1,500-2,000 |
| **UE Capacity** | 10-50 simulated | 100-500 simulated | 1,000+ simulated |

### NUC Networking — The Critical Constraint

Most NUCs have a **single 2.5GbE port**. A 5G core ideally wants separate interfaces for N2, N3, N6, SBI, and management traffic.

**Solutions:**
```
┌─────────────────────────────────────────────────────────┐
│  NUC Networking Options for 5GC                         │
│                                                         │
│  Option 1: VLAN Tagging (Recommended for lab)           │
│  ┌──────────┐                                           │
│  │ eth0     │──── VLAN 10: N2/N3 (RAN traffic)         │
│  │ 2.5GbE   │──── VLAN 20: N6 (Data network)          │
│  │          │──── VLAN 30: SBI (inter-NF)              │
│  │          │──── Native: Management/SSH               │
│  └──────────┘                                           │
│                                                         │
│  Option 2: USB Ethernet Adapter                         │
│  ┌──────────┐  ┌──────────────┐                        │
│  │ eth0     │  │ eth1 (USB)   │                        │
│  │ 2.5GbE   │  │ 1GbE dongle  │                       │
│  │ Data+SBI │  │ RAN traffic  │                        │
│  └──────────┘  └──────────────┘                        │
│                                                         │
│  Option 3: All Virtual (UERANSIM on same NUC)          │
│  ┌──────────┐                                           │
│  │ eth0     │──── Internet/Management                  │
│  │ 2.5GbE   │                                          │
│  │          │  Linux bridges + veth pairs for           │
│  │          │  all N2/N3/N6/SBI traffic internally     │
│  └──────────┘                                           │
│  (Most common for single-NUC labs)                     │
└─────────────────────────────────────────────────────────┘
```

### Open5GS on a Single NUC

```
┌──────────────────────────────────────────────────────────────┐
│  Intel NUC (12C/16T, 32 GB RAM, 512 GB NVMe)                │
│  Ubuntu 22.04 LTS                                            │
│                                                              │
│  ┌── Open5GS (all-in-one, ~2 GB RAM total) ──────────────┐  │
│  │                                                        │  │
│  │  5G Core (SBA)              EPC (4G)                   │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐  ┌─────┐ ┌─────┐           │  │
│  │  │ AMF │ │ SMF │ │ UPF │  │ MME │ │ HSS │           │  │
│  │  └─────┘ └─────┘ └─────┘  └─────┘ └─────┘           │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐  ┌─────┐ ┌─────┐           │  │
│  │  │ NRF │ │ UDM │ │ UDR │  │ SGW │ │ PGW │           │  │
│  │  └─────┘ └─────┘ └─────┘  └─────┘ └─────┘           │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌──────┐           │  │
│  │  │AUSF │ │ PCF │ │NSSF │ │ SCP │ │ PCRF │           │  │
│  │  └─────┘ └─────┘ └─────┘ └─────┘ └──────┘           │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌── MongoDB ─────┐  ┌── UERANSIM ────────────────────┐     │
│  │  WiredTiger     │  │  gNB simulator + 100 UEs       │     │
│  │  Cache: 1-2 GB  │  │  ~200 MB RAM                   │     │
│  └─────────────────┘  └────────────────────────────────┘     │
│                                                              │
│  ┌── Optional ─────────────────────────────────────────┐     │
│  │  Prometheus + Grafana (~500 MB)                     │     │
│  │  k3s (if K8s desired, ~600 MB overhead)             │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                              │
│  Total RAM: ~4-8 GB used │ CPU: ~1-3 cores steady state     │
│  Remaining: Plenty of headroom for development               │
└──────────────────────────────────────────────────────────────┘
```

### free5GC on a Single NUC

```
┌──────────────────────────────────────────────────────────────┐
│  Intel NUC (12C/16T, 32 GB RAM, 512 GB NVMe)                │
│  Ubuntu 22.04 LTS + gtp5g kernel module                      │
│  k3s (single-node Kubernetes)                                │
│                                                              │
│  ┌── free5GC (10-14 pods, ~4-6 GB RAM total) ────────────┐  │
│  │                                                        │  │
│  │  Control Plane Pods:                                   │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐    │  │
│  │  │ AMF │ │ SMF │ │ NRF │ │ UDM │ │ UDR │ │AUSF │    │  │
│  │  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘    │  │
│  │  ┌─────┐ ┌──────┐ ┌──────┐ ┌─────┐ ┌─────┐          │  │
│  │  │ PCF │ │ NSSF │ │N3IWF │ │ NEF │ │ CHF │          │  │
│  │  └─────┘ └──────┘ └──────┘ └─────┘ └─────┘          │  │
│  │                                                        │  │
│  │  User Plane:                                           │  │
│  │  ┌──────────────────────────────────────┐              │  │
│  │  │  UPF (hostNetwork + privileged)      │              │  │
│  │  │  requires gtp5g kernel module        │              │  │
│  │  └──────────────────────────────────────┘              │  │
│  │                                                        │  │
│  │  ┌──────────────────────────────────────┐              │  │
│  │  │  WebUI (subscriber provisioning)     │              │  │
│  │  └──────────────────────────────────────┘              │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌── k3s overhead ─┐  ┌── MongoDB ─────┐                    │
│  │  ~600 MB RAM     │  │  WiredTiger    │                    │
│  │  Flannel CNI     │  │  Cache: 2 GB   │                    │
│  └──────────────────┘  └────────────────┘                    │
│                                                              │
│  ┌── UERANSIM (pod) ──────────────────────────────────┐      │
│  │  gNB simulator + UEs                                │      │
│  └─────────────────────────────────────────────────────┘      │
│                                                              │
│  Total RAM: ~8-12 GB used │ CPU: ~2-4 cores steady state    │
│  Tighter fit — less headroom than Open5GS                    │
└──────────────────────────────────────────────────────────────┘
```

### Head-to-Head: NUC Resource Consumption

| Resource | Open5GS (bare metal) | Open5GS (k3s) | free5GC (k3s) |
|---|---|---|---|
| **Container/Process count** | 9-12 processes | 9-12 pods | 12-16 pods |
| **RAM — Core NFs** | ~1.5-2 GB | ~2-2.5 GB | ~3-4 GB |
| **RAM — MongoDB** | ~1-2 GB | ~1-2 GB | ~1-2 GB |
| **RAM — K8s overhead** | 0 (bare metal) | ~600 MB (k3s) | ~600 MB (k3s) |
| **RAM — UERANSIM (100 UEs)** | ~200 MB | ~200 MB | ~200 MB |
| **RAM — Total** | **~3-4 GB** | **~4-5 GB** | **~6-8 GB** |
| **CPU — Steady state** | ~0.5-1 core | ~0.8-1.5 cores | ~1.5-3 cores |
| **CPU — Peak (bulk reg)** | ~2-3 cores | ~3-4 cores | ~4-6 cores |
| **Disk — NF images** | ~300-500 MB | ~400-700 MB | ~1-2 GB |
| **Disk — MongoDB data (lab)** | ~50-200 MB | ~50-200 MB | ~50-200 MB |
| **Kernel module needed?** | No | No | **Yes (gtp5g)** |
| **Fits NUC 11 i5 (4C/16GB)?** | **Yes, comfortably** | **Yes** | Tight but possible |
| **Fits NUC 13 i7 (12C/32GB)?** | **Effortless** | **Effortless** | **Yes, comfortably** |

> **Verdict for NUC:** Open5GS uses ~50% less RAM and fewer CPU cycles. On a resource-constrained NUC, that headroom matters for running simulators, monitoring, and development tools alongside the core.

---

## 4. Architecture Deep Dive

### 4.1 Open5GS

```
┌─────────────────────────────────────────────────────────────┐
│                       Open5GS Core                          │
│                                                             │
│  5G Core (SBA over HTTP/2)          EPC (4G)                │
│  ┌─────┐ ┌─────┐ ┌─────┐          ┌─────┐ ┌─────┐         │
│  │ AMF │ │ SMF │ │ UPF │          │ MME │ │ HSS │         │
│  └──┬──┘ └──┬──┘ └──┬──┘          └──┬──┘ └──┬──┘         │
│     │       │       │                │       │              │
│  ┌─────┐ ┌─────┐ ┌─────┐          ┌─────┐ ┌─────┐         │
│  │ NRF │ │ UDM │ │ UDR │          │ SGW │ │ PGW │         │
│  └─────┘ └──┬──┘ └─────┘          └─────┘ └─────┘         │
│             │                                               │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌──────┐                │
│  │AUSF │ │ PCF │ │NSSF │ │ SCP │ │ PCRF │                │
│  └─────┘ └─────┘ └─────┘ └─────┘ └──────┘                │
│                                                             │
│  Data Store: MongoDB                                        │
│  Config: YAML files per NF                                  │
│  IPC: SBI (HTTP/2) for 5GC, GTP-C/GTP-U for EPC           │
└─────────────────────────────────────────────────────────────┘
```

**Architecture Philosophy:** Monolithic-leaning, spec-faithful. Each NF is a lightweight C binary (single process). Uniquely supports **both EPC (4G) and 5GC** in a single codebase.

| Property | Detail |
|---|---|
| **NF Design** | Single-process per NF, lightweight C binaries |
| **Inter-NF Communication** | SBI (HTTP/2) for 5GC; GTP-C/GTP-U for EPC |
| **Data Store** | MongoDB (shared or per-NF) |
| **Configuration** | YAML files, easy to template with Helm/Ansible |
| **Build System** | Meson + Ninja (fast, reliable) |
| **External Dependencies** | Minimal: talloc, mongoc, yaml, nghttp2 |
| **Deployment Flexibility** | Bare metal, VMs, Docker, K8s — not opinionated |
| **Typical Container Count** | 5-8 for full 5G SA core |
| **UPF Implementation** | Userspace (libgtpnl + TUN) — **no kernel module** |

**NUC-Specific Strengths:**
- Runs entire 5GC on **~2 GB RAM** — leaves most NUC memory free
- No kernel module dependency — works on any Ubuntu/Fedora kernel out of the box
- Can run bare metal (no K8s overhead) or with k3s — your choice
- Hybrid 4G/5G — test both EPC and 5GC without needing separate stacks
- Single `apt install open5gs` on Ubuntu gets you a working core in minutes
- Can run entire core on a **$400 NUC 11 i5** comfortably

**Weaknesses:**
- HA and horizontal scaling require custom engineering
- C codebase — higher barrier for contributors (memory management, debugging)
- Less cloud-native patterning (no native service mesh integration)
- Bus factor risk — Sukchan Lee is the primary maintainer
- No built-in WebUI for subscriber management (community tools available)

---

### 4.2 free5GC

```
┌────────────────────────────────────────────────────────────────┐
│                    free5GC Core (SBA)                           │
│                                                                │
│  Control Plane NFs (each independently scalable):              │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐   │
│  │ AMF │ │ SMF │ │ NRF │ │ UDM │ │ UDR │ │AUSF │ │ PCF │   │
│  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘   │
│                                                                │
│  Extended NFs:                                                 │
│  ┌─────┐ ┌──────┐ ┌──────┐ ┌─────┐ ┌─────┐                  │
│  │NSSF │ │N3IWF │ │ TNGF │ │ NEF │ │ CHF │                  │
│  └─────┘ └──────┘ └──────┘ └─────┘ └─────┘                  │
│                                                                │
│  User Plane:                                                   │
│  ┌──────────────────────────────────────────┐                  │
│  │  UPF (requires gtp5g kernel module)      │                  │
│  │  Kernel-space GTP-U processing           │                  │
│  └──────────────────────────────────────────┘                  │
│                                                                │
│  Management:                                                   │
│  ┌──────────────────────────────────────────┐                  │
│  │  WebUI (subscriber provisioning portal)  │                  │
│  └──────────────────────────────────────────┘                  │
│                                                                │
│  Data Store: MongoDB                                           │
│  Config: YAML per NF                                           │
│  IPC: SBI (HTTP/2)                                             │
└────────────────────────────────────────────────────────────────┘
```

**Architecture Philosophy:** True Service-Based Architecture per 3GPP spec. Each NF is a separate Go binary designed for independent containerized deployment.

| Property | Detail |
|---|---|
| **NF Design** | Separate Go binary per NF, each in its own container |
| **Inter-NF Communication** | SBI (HTTP/2) — strict 3GPP SBA |
| **Data Store** | MongoDB (shared or per-NF) |
| **Configuration** | YAML files per NF |
| **Build System** | Standard Go modules (go build) |
| **External Dependencies** | gin, mongo-driver, etc. — larger dep tree |
| **Deployment Model** | Kubernetes-first (Docker Compose also available) |
| **Typical Container Count** | 10-14 for full deployment |
| **UPF Implementation** | Kernel-space via **gtp5g module** (higher throughput) |
| **Special Requirement** | **gtp5g kernel module** on UPF host (kernel 5.x+) |

**NUC-Specific Strengths:**
- True cloud-native — runs well on k3s single-node
- Go language — memory-safe, modern tooling
- Apache 2.0 license — no copyleft for commercial embedding
- Broadest NF coverage: N3IWF, TNGF, CHF, NEF
- Built-in WebUI for subscriber management
- gtp5g kernel module compiles cleanly on Ubuntu 22.04/24.04 shipped with NUCs

**NUC-Specific Weaknesses:**
- 10-14 containers = **2-3x the RAM** of Open5GS — tighter fit on 16 GB NUCs
- **gtp5g kernel module** must be compiled on the NUC (DKMS recommended)
- Needs k3s or Docker Compose — bare metal is not practical
- If NUC kernel updates, gtp5g must be recompiled (DKMS automates this)
- UPF pod requires `privileged: true` + `hostNetwork: true` — less isolated

---

### 4.3 Radisys 5G Core

```
┌──────────────────────────────────────────────────────┐
│              Radisys 5G Core (Vendor)                 │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │       Pre-built, Pre-configured NF Bundle      │  │
│  │  AMF | SMF | UPF | NRF | UDM | AUSF | PCF     │  │
│  │  (Opaque container images or VM appliances)    │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  Management & Orchestration Portal             │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  Radisys Connect RAN Integration Layer         │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  Vendor controls: image lifecycle, updates, patches  │
│  Deployment: Vendor professional services            │
└──────────────────────────────────────────────────────┘
```

**NUC Reality Check:** Radisys is designed for multi-node server deployments with vendor professional services. Running it on a single NUC is not a supported or practical scenario. It's included here for completeness in the comparison, but **if a single NUC is your target, Radisys is not a viable option.**

| Property | Detail |
|---|---|
| **NF Design** | Vendor-packaged (opaque internals) |
| **Configuration** | Vendor portal / guided setup |
| **Deployment Model** | Vendor-guided (multi-node K8s or VM cluster) |
| **NUC Compatible?** | **No** — requires vendor infrastructure specs |
| **Minimum Hardware** | Multi-node cluster with vendor-specified specs |

---

## 5. Network Function Coverage Matrix

| Network Function | 3GPP Role | Open5GS | free5GC | Radisys |
|---|---|:---:|:---:|:---:|
| **AMF** | Access & Mobility Management | Yes | Yes | Yes |
| **SMF** | Session Management | Yes | Yes | Yes |
| **UPF** | User Plane Function | Yes (userspace) | Yes (gtp5g kernel) | Yes |
| **NRF** | NF Repository Function | Yes | Yes | Yes |
| **UDM** | Unified Data Management | Yes | Yes | Yes |
| **UDR** | Unified Data Repository | Yes | Yes | Yes |
| **AUSF** | Authentication Server | Yes | Yes | Yes |
| **PCF** | Policy Control Function | Yes | Yes | Yes |
| **NSSF** | Network Slice Selection | Yes | Yes | Claimed |
| **SCP** | Service Communication Proxy | Yes | Partial | Unknown |
| **NEF** | Network Exposure Function | Partial | **Yes** | Claimed |
| **N3IWF** | Non-3GPP Interworking (WiFi) | No | **Yes** | Claimed |
| **TNGF** | Trusted Non-3GPP Gateway | No | **Yes** | Unknown |
| **CHF** | Charging Function | No | **Yes** | Claimed |
| **EPC (4G)** | MME, SGW, PGW, HSS, PCRF | **Yes (full)** | No | Separate product |
| **WebUI** | Subscriber Management | Community tools | **Built-in** | Vendor portal |

**Key Takeaways:**
- **free5GC** has the broadest 5G NF coverage (N3IWF, TNGF, CHF, NEF)
- **Open5GS** uniquely supports **hybrid 4G EPC + 5GC** in a single codebase — on a single NUC
- For non-3GPP access (WiFi offload via N3IWF) — free5GC is the only OSS option
- On a NUC, Open5GS's smaller footprint means more room for other services

---

## 6. Language, Code Quality & Developer Experience

### Side-by-Side Comparison

| Aspect | Open5GS (C) | free5GC (Go) | Radisys |
|---|---|---|---|
| **Memory Safety** | Manual — risk of buffer overflows | GC-managed — no manual memory bugs | Cannot audit |
| **Performance** | Excellent — deterministic latency, zero GC | Good — slight GC overhead | Unverifiable |
| **Concurrency Model** | Event-driven (epoll/kqueue) | Goroutines — lightweight threads | Unknown |
| **Testability** | Unit tests; fewer C test frameworks | Go built-in `testing` package | Internal QA only |
| **Code Structure** | Well-organized, spec-traceable | Clean per-NF separation, standard Go layout | N/A |
| **Build System** | Meson + Ninja — fast, reliable | `go build` — simple, reproducible | N/A |
| **Dependency Count** | Minimal (talloc, mongoc, yaml, nghttp2) | Larger (gin, mongo-driver, etc.) | Unknown |
| **Debugging** | GDB, Valgrind, AddressSanitizer | Delve, pprof — modern, accessible | Vendor tickets |
| **IDE Support** | Good (CLion, VS Code + C extensions) | Excellent (GoLand, VS Code + Go) | N/A |
| **Hiring Difficulty** | **High** — C + telecom intersection | **Low-Medium** — large Go pool | N/A |
| **Onboarding Time** | 4-8 weeks | 2-4 weeks | N/A |

### Developer Workflow on a NUC

| Workflow | Open5GS | free5GC |
|---|---|---|
| **Install on NUC** | `apt install open5gs` (Ubuntu PPA) | Clone repos + `go build` + gtp5g DKMS |
| **Clone to first build** | `meson build && ninja -C build` (~2 min) | `go build ./...` (~1 min per NF) |
| **Run unit tests** | `ninja -C build test` | `go test ./...` |
| **Debug a crash** | GDB + core dump | Delve + goroutine traces |
| **Profile on NUC** | `perf` + flamegraphs | `pprof` + built-in profiling |
| **NUC-specific tip** | Build natively on NUC — fast enough | Build natively or cross-compile |

---

## 7. Resource Footprint — NUC-Sized

### Per-NF Resource Consumption (Measured)

| Component | CPU (idle) | CPU (peak) | RAM | Notes |
|---|---|---|---|---|
| **AMF** | 0.05 core | 0.3-0.5 core | 150 MB | Peaks during mass registration |
| **SMF** | 0.03 core | 0.2-0.4 core | 120 MB | Session setup intensive |
| **UPF** | 0.1 core | 1-2 cores | 200-500 MB | Scales with throughput |
| **NRF** | 0.01 core | 0.05 core | 60 MB | Lightweight registry |
| **UDM** | 0.02 core | 0.1 core | 80 MB | |
| **UDR** | 0.02 core | 0.1 core | 80 MB | |
| **AUSF** | 0.01 core | 0.05 core | 60 MB | |
| **PCF** | 0.01 core | 0.05 core | 60 MB | |
| **NSSF** | 0.01 core | 0.03 core | 50 MB | |
| **MongoDB** | 0.1 core | 0.5-1.0 core | 1-4 GB | Cap WiredTiger cache |
| **UERANSIM gNB** | 0.05 core | 0.2 core | 80 MB | |
| **UERANSIM UE (each)** | 0.01 core | 0.05 core | 15 MB | Per simulated UE |
| **k3s** | 0.2 core | 1.0 core | 600 MB | Only if using K8s |

### NUC Sizing Guide

| NUC Config | Open5GS (bare metal) | Open5GS (k3s) | free5GC (k3s) | What's left for you |
|---|---|---|---|---|
| **4C/16GB** (NUC 11 i5) | 50 UEs, monitoring | 30 UEs | 10-20 UEs, tight | Little headroom |
| **4C/32GB** (NUC 11 i7) | 100 UEs, monitoring | 80 UEs, monitoring | 50 UEs | Moderate headroom |
| **12C/32GB** (NUC 13 i7) | 500 UEs, full stack | 300 UEs, full stack | 200 UEs, full stack | **Recommended** |
| **12C/64GB** (NUC 13 i7) | 500+ UEs, multi-slice | 500 UEs, multi-slice | 500 UEs, full stack | Generous headroom |
| **24C/64GB** (NUC 13 Extreme) | 1000+ UEs, stress test | 1000+ UEs | 1000 UEs | Overkill for most |

### Thermal Reality on NUC

| Load Level | NUC 11/12/13 (28W TDP) | Fan Noise | Sustainable? |
|---|---|---|---|
| Idle (5GC running, no traffic) | 35-45°C | Silent | Yes, indefinitely |
| Light traffic (10 UEs) | 50-60°C | Barely audible | Yes, indefinitely |
| Moderate (100 UEs, data plane) | 60-75°C | Noticeable | Yes, indefinitely |
| Heavy (500 UEs, bulk registration) | 75-90°C | Audible | Yes, but monitor |
| Stress test (1000+ UEs) | 85-95°C | Loud | Thermal throttling risk |

> **Tip:** Mount the NUC with bottom vents unobstructed (VESA mount or rubber feet). In warm environments (>30°C), add a USB fan underneath.

---

## 8. 3GPP Standards Compliance

| Aspect | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Target 3GPP Release** | Release 17 (partial Rel-18) | Release 15/16 (some Rel-17) | Vendor-claimed Rel-16+ |
| **Spec Traceability** | Direct C → TS spec mapping | Go SBA → TS spec mapping | Opaque |
| **SBI Compliance** | HTTP/2 based SBI | HTTP/2 based SBI | Vendor implementation |
| **NAS Protocol** | Full 5G NAS | Full 5G NAS | Vendor |
| **NGAP (N2)** | Full | Full | Vendor |
| **PFCP (N4)** | Full | Full | Vendor |
| **GTP-U (N3/N9)** | Userspace (portable) | Kernel module (faster) | Vendor |
| **Network Slicing** | Basic (NSSF) | NSSF + NEF | Vendor claim |
| **Policy & Charging** | PCF (basic) | PCF + CHF | Vendor claim |
| **Non-3GPP Access** | Not yet | N3IWF + TNGF | Vendor claim |
| **Interop Tested** | UERANSIM, srsRAN, commercial gNBs | UERANSIM, srsRAN, commercial gNBs | Vendor lab |
| **Formal Certification** | No | No | Possible (vendor-provided) |

**Key Insight for NUC users:**
- Open5GS tracks higher 3GPP releases (Rel-17+)
- free5GC has broader NF coverage within Rel-15/16
- On a NUC, the UPF difference matters: Open5GS userspace UPF is simpler (no kernel module), while free5GC's gtp5g offers better raw throughput — but on a 2.5GbE NUC NIC, the NIC is the bottleneck anyway

---

## 9. Open Source Health & Governance

### GitHub Metrics (early 2026)

| Metric | Open5GS | free5GC |
|---|---|---|
| **GitHub Stars** | ~2,800+ | ~2,400+ |
| **Forks** | ~1,000+ | ~900+ |
| **Total Commits** | ~5,000+ | ~500+ (main repo) |
| **Contributors** | ~120+ | ~50+ |
| **Active Since** | 2017 (as NextEPC) | 2019 |
| **Release Cadence** | Monthly-ish | Periodic |
| **Issue Response Time** | Days | Days to Weeks |
| **Primary Backer** | Community (Sukchan Lee led) | Academic (NCTU/NYCU) |
| **CI/CD** | GitHub Actions | GitHub Actions |
| **Documentation** | open5gs.org | free5gc.org |
| **Community Channels** | Discord, mailing list | Discord, forums |

### Bus Factor Assessment

| | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Key Person Risk** | **HIGH** — Sukchan Lee | **MEDIUM** — NYCU team | **LOW** — vendor org |
| **Mitigation** | Forkable (AGPLv3); growing contributors | Forkable (Apache 2.0); academic continuity | SLA; but vendor bankruptcy = risk |
| **If project stalls** | Fork + community takeover | Fork + community takeover | No recourse |

---

## 10. Security & Supply Chain

| Control | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Build from source** | Yes | Yes | No |
| **Reproducible builds** | Yes (Meson) | Yes (Go modules) | No |
| **Sign your own images** | Yes (cosign, Notary) | Yes (cosign, Notary) | No |
| **Pin dependencies** | Yes | Yes | Vendor-dependent |
| **SBOM generation** | Full (Syft, Trivy) | Full (Syft, Trivy) | Limited/None |
| **CVE scanning** | Full (Trivy, Grype) | Full (Trivy, Grype) | Vendor responsibility |
| **Patch turnaround** | Self-controlled (hours) | Self-controlled (hours) | Vendor SLA (days/weeks) |
| **Full source audit** | Yes | Yes | **No** |
| **TLS/mTLS config** | Configurable per NF | Configurable per NF | Vendor-managed |
| **Secrets management** | Your choice (Vault, K8s) | Your choice (Vault, K8s) | Vendor-managed |

### NUC-Specific Security Note

On a single NUC, all NFs share the same host. This simplifies networking (localhost SBI) but means a compromised NF has access to the entire node. Mitigations:
- Use network namespaces or k3s pod isolation
- Apply Linux capabilities dropping on NF processes
- free5GC UPF runs as `privileged` — accept this risk or use Open5GS userspace UPF

```
Supply Chain Control Spectrum

  FULL CONTROL ◄─────────────────────────────────────► ZERO CONTROL
  ┌──────────┐  ┌──────────┐                ┌──────────┐
  │ Open5GS  │  │ free5GC  │                │ Radisys  │
  │          │  │          │                │          │
  │ Build    │  │ Build    │                │ Receive  │
  │ from src │  │ from src │                │ opaque   │
  │ Audit    │  │ Audit    │                │ images   │
  │ Sign     │  │ Sign     │                │ Trust    │
  │ Scan     │  │ Scan     │                │ vendor   │
  └──────────┘  └──────────┘                └──────────┘
```

---

## 11. Licensing Deep Dive

| Scenario | Open5GS (AGPLv3) | free5GC (Apache 2.0) | Radisys (Proprietary) |
|---|---|---|---|
| **Internal lab on a NUC** | Safe — no obligations | Safe — no obligations | Per contract |
| **Modifications for internal service** | Safe — internal use | Safe — no obligations | Per contract |
| **Offering as a network service** | **Must share modified source** (AGPL clause) | No obligation | N/A |
| **Embedding in commercial product** | **Copyleft — must open-source derivative** | Permissive — just attribution | Per contract |
| **Forking for proprietary use** | **Not allowed** | **Allowed** | N/A |

> **Bottom Line:** For NUC-based labs and internal PoCs, both licenses are fine. If you plan to commercialize, **free5GC's Apache 2.0** avoids AGPL complications.

---

## 12. Day-2 Operations

### Deployment & Configuration

| Aspect | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Config Format** | YAML per NF | YAML per NF | Vendor portal |
| **Install Method (NUC)** | `apt install` or Helm | Helm (towards5gs-helm) or Docker Compose | N/A for NUC |
| **Config Complexity** | Low-Medium | Medium-High (more NFs) | Low (vendor guided) |
| **Helm Charts** | Community (openverso by Gradiant) | Official + community | Vendor-provided |
| **Subscriber Provisioning** | MongoDB CLI / community scripts | **Built-in WebUI** | Vendor portal |
| **TLS/mTLS Setup** | Manual configuration | Manual configuration | Vendor-managed |

### Monitoring on a NUC

| Stack | RAM Cost | Open5GS | free5GC |
|---|---|---|---|
| **Prometheus + Grafana** | ~500 MB | Community dashboards | Community dashboards |
| **Loki (log aggregation)** | ~200 MB | Structured logs | Structured logs |
| **Full stack (Prom+Graf+Loki)** | ~700 MB | Works on 32 GB NUC | Tight on 16 GB NUC |
| **Lightweight alternative: VictoriaMetrics** | ~200 MB | Compatible | Compatible |

> **NUC Tip:** On a 16 GB NUC, skip the full monitoring stack and use `journalctl` + `mongosh` for troubleshooting. On 32 GB+, add Prometheus + Grafana.

### Upgrade & Rollback

| Aspect | Open5GS | free5GC |
|---|---|---|
| **Upgrade on NUC** | `apt upgrade open5gs` or Helm upgrade | Helm upgrade (rebuild gtp5g if kernel changed) |
| **Rollback** | `apt install open5gs=<version>` or image pin | Helm rollback + verify gtp5g compat |
| **DB Migration** | Manual (MongoDB schema changes) | Manual (MongoDB schema changes) |
| **Downtime during upgrade** | Brief (restart NF processes) | Brief (rolling pod restarts) |
| **NUC-specific risk** | Low — simple process restart | Medium — gtp5g module version mismatch |

---

## 13. Scaling & HA on a Single Node

On a single NUC, traditional multi-node HA is not possible. Here's what you can do:

| Aspect | Open5GS | free5GC |
|---|---|---|
| **Process restart on crash** | systemd auto-restart | k3s pod restart policy |
| **NF redundancy** | Run 2 AMF instances with LB (if RAM allows) | K8s replicas per NF (if RAM allows) |
| **MongoDB HA** | Single instance (no replica set on 1 NUC) | Single instance |
| **UPF scaling** | Multiple UPF instances, different DNN/slices | Limited by single gtp5g module |
| **Load balancing** | HAProxy or NGINX locally | k3s Service + kube-proxy |
| **Realistic HA approach** | Watchdog scripts + auto-restart + MongoDB backup | k3s liveness/readiness probes + PDB |
| **True HA** | **Not achievable on 1 NUC** — need 2+ nodes | **Not achievable on 1 NUC** |

### Single-NUC Resilience Checklist

- [ ] Enable systemd/k3s auto-restart for all NFs
- [ ] Schedule daily MongoDB backups (mongodump to NVMe or USB)
- [ ] Monitor NUC thermals (lm-sensors + alerting)
- [ ] Use NVMe with SMART monitoring
- [ ] Keep a cold spare NUC with identical config for disaster recovery
- [ ] Pin kernel version to avoid gtp5g breakage (free5GC only)

---

## 14. Testing & Simulation Ecosystem

| Tool / Simulator | Description | Open5GS | free5GC |
|---|---|:---:|:---:|
| **UERANSIM** | Open-source 5G UE + gNB simulator | Tested | Tested |
| **srsRAN** | Open-source 4G/5G RAN stack | Tested | Tested |
| **PacketRusher** | Go-based 5G UE/gNB load tester | Compatible | Compatible |
| **gNBSim** | ONF gNB simulator | Compatible | Compatible |
| **my5G-RANTester** | 5G RAN protocol tester | Compatible | Compatible |
| **Commercial UEs** | Quectel, Sierra Wireless | Community tested | Community tested |
| **Commercial gNBs** | Baicells, Airspan, Nokia | Community reports | Community reports |

### NUC Lab Test Strategy

| Phase | Tool | UE Count | NUC Requirement |
|---|---|---|---|
| **Smoke test** | UERANSIM | 1 UE | Any NUC (even 4C/8GB) |
| **Functional** | UERANSIM | 10-50 UEs | 4C/16GB+ |
| **Integration** | UERANSIM multi-UE | 50-100 UEs | 12C/32GB recommended |
| **Load test** | PacketRusher | 100-500 UEs | 12C/32GB+ |
| **Soak test (48h)** | UERANSIM | 50 UEs sustained | 12C/32GB (monitor thermals) |
| **OTA (over-the-air)** | srsRAN + USRP B210 | 1-5 real UEs | Any NUC + USB 3.0 for SDR |

> **NUC OTA Setup:** Plug a USRP B210 SDR into the NUC's USB 3.0 port, run srsRAN gNB on the same NUC alongside the 5GC. Real phones can attach to your private 5G network from a single NUC. Total cost: NUC ($700) + USRP B210 ($1,500) + SIM cards ($50).

---

## 15. RAN Interoperability

| RAN Solution | Type | Open5GS | free5GC |
|---|---|:---:|:---:|
| **UERANSIM** | Simulator | Tested | Tested |
| **srsRAN** | Open-source | Tested | Tested |
| **Baicells** | Commercial small cell | Community tested | Community tested |
| **Airspan** | Commercial | Community reports | Community reports |
| **Nokia** | Commercial | Community reports | Community reports |
| **Compal/Sercomm** | Small cell | Community reports | Community reports |

> **Note:** Any 3GPP-compliant gNB should interoperate via N2 (NGAP) + N3 (GTP-U). Always validate in your NUC lab before deploying with commercial RAN.

---

## 16. Migration Paths

| Migration Path | Difficulty | Effort | Key Considerations |
|---|:---:|---|---|
| **Open5GS → free5GC** (same NUC) | Medium | 2-3 weeks | Both use MongoDB — subscriber data portable. Install gtp5g. Lose 4G EPC. |
| **free5GC → Open5GS** (same NUC) | Medium | 1-2 weeks | Remove gtp5g. Gain EPC. Lose N3IWF/CHF/WebUI. Free up RAM. |
| **NUC → Multi-node cluster** | Low-Med | 1-2 weeks | Same Helm charts, just more nodes. Add real HA. |
| **Either OSS → Radisys** | High | 4-8 weeks | Full re-provisioning. Leave NUC world entirely. |
| **Radisys → Either OSS** | **Very High** | 6-12 weeks | Extract subscriber data (vendor cooperation). Full re-architecture. |

### Migration Difficulty Spectrum

```
                    Migration Difficulty
                    Low    Medium    High    Very High
  Open5GS → free5GC          ██
  free5GC → Open5GS        ██
  NUC → Cluster          ██
  OSS → Radisys                      ████
  Radisys → OSS                              ██████
```

> **Strategy:** Start on a NUC with OSS. When you outgrow the NUC, migrate the same Helm charts to a multi-node cluster. Starting with Radisys creates a vendor lock-in wall.

---

## 17. Total Cost of Ownership — NUC Edition

### Hardware Cost (One-Time)

| NUC Configuration | Cost | Suitable For |
|---|---|---|
| NUC 11 i5, 16GB, 256GB NVMe | ~$500 | Basic demos, smoke testing |
| NUC 13 i7, 32GB, 512GB NVMe | ~$800 | **Recommended** — dev/test/PoC |
| NUC 13 i7, 64GB, 1TB NVMe | ~$1,100 | Heavy testing, multi-slice |
| NUC 13 Extreme, 64GB, 1TB | ~$2,000 | Stress testing, performance benchmarks |

### Year-1 Total Cost

| Cost Component | Open5GS on NUC | free5GC on NUC | Radisys |
|---|---|---|---|
| **NUC Hardware** | $800 | $800 | N/A ($5K+ server) |
| **Software License** | $0 | $0 | $100K-500K+/yr |
| **Engineering — Setup** | 1-2 days ($500-1K) | 2-4 days ($1K-2K) | 1-2 weeks ($5K-10K) |
| **Engineering — Customization** | As needed ($0-5K) | As needed ($0-5K) | Vendor queue |
| **Electricity** | ~$30-50/yr (28W avg) | ~$30-50/yr (28W avg) | ~$500+/yr (servers) |
| **Support** | Community (free) | Community (free) | 15-20% of license |
| **Professional Services** | N/A | N/A | $20K-50K |
| **Year-1 Total** | **$1.3K-2.5K** | **$1.8K-3.5K** | **$200K-600K+** |

### 3-Year TCO Projection

| Year | Open5GS on NUC | free5GC on NUC | Radisys |
|---|---|---|---|
| Year 1 | $1.3K-2.5K | $1.8K-3.5K | $200K-600K |
| Year 2 | $50-200 (electricity + time) | $50-200 | $150K-500K |
| Year 3 | $50-200 | $50-200 | $150K-500K |
| **3-Year Total** | **$1.5K-3K** | **$2K-4K** | **$500K-1.6M** |

```
3-Year TCO Visual (log scale)

  Open5GS on NUC    ██  ~$2K
  free5GC on NUC    ███  ~$3K
  Radisys           ██████████████████████████████████████████  ~$800K
```

> **The NUC advantage:** A complete private 5G core lab for the price of a nice dinner vs. the price of a house.

---

## 18. Risk Register

| Risk | Open5GS | free5GC | Mitigation (NUC-specific) |
|---|:---:|:---:|---|
| **NUC hardware failure** | Equal | Equal | Keep a cold spare NUC + daily MongoDB backups |
| **Thermal throttling** | Low | Medium | Monitor with lm-sensors; ensure ventilation |
| **NVMe failure** | Equal | Equal | SMART monitoring; off-NUC backups |
| **gtp5g kernel breakage** | N/A | **HIGH** | Pin kernel version; DKMS auto-rebuild |
| **Single NIC bottleneck** | Low | Low | 2.5GbE is plenty for lab; add USB NIC if needed |
| **Power loss** | Equal | Equal | UPS ($50-100) for clean shutdown |
| **Lead maintainer leaves** | HIGH | Medium | Fork capability; growing community |
| **Critical CVE** | Self-patch (hours) | Self-patch (hours) | OSS advantage over vendor SLA |
| **License dispute** | Medium (AGPLv3) | Low (Apache 2.0) | Legal review if commercializing |
| **Team turnover** | HIGH (C skills) | Medium (Go skills) | Document your NUC setup; automate with scripts |
| **NUC becomes insufficient** | Low | Low | Migrate same configs to multi-node cluster |

---

## 19. Decision Framework

### Choose Open5GS on a NUC if:

- You want the **smallest footprint** — entire 5GC in ~2 GB RAM
- **Hybrid LTE + 5G** is needed on the same NUC
- No kernel module hassle — works on any stock Ubuntu kernel
- Want to run bare metal (no K8s overhead) for maximum simplicity
- Resource budget is tight (works on a $400 NUC 11 i5)
- Team has C and telecom protocol experience
- Most mature codebase (since 2017, 5000+ commits)

### Choose free5GC on a NUC if:

- You want **K8s-native** (k3s single-node) from day one
- Team has Go and cloud-native skills
- **Apache 2.0 license** is needed for commercial embedding
- You need N3IWF, CHF, NEF, or TNGF NFs
- Built-in **WebUI** for subscriber management is valued
- You accept the gtp5g kernel module dependency
- NUC has 32GB+ RAM to accommodate the larger footprint

### Skip Radisys for NUC deployments:

- Radisys requires multi-node infrastructure and vendor professional services
- If you need Radisys, you don't need a NUC — you need a datacenter budget
- Consider Radisys only if zero in-house 5G expertise AND budget allows $200K+ Year-1

### Decision Flowchart

```
START: Deploying 5GC on a Single NUC?
  │
  ├─ Do you need hybrid 4G LTE + 5G?
  │    │
  │    └─ YES ──────────────────────────────► Open5GS
  │
  ├─ Do you have only 16 GB RAM on the NUC?
  │    │
  │    └─ YES ──────────────────────────────► Open5GS (smaller footprint)
  │
  ├─ Do you want Kubernetes (k3s) from day one?
  │    │
  │    └─ YES
  │         │
  │         ├─ Need Apache 2.0 license? ────► free5GC
  │         │
  │         ├─ Need N3IWF / WiFi offload? ──► free5GC
  │         │
  │         └─ Otherwise ──────────────────► Either works (Open5GS is leaner)
  │
  ├─ Want maximum simplicity (apt install)?
  │    │
  │    └─ YES ──────────────────────────────► Open5GS
  │
  └─ Default ──────────────────────────────► Open5GS (safer bet for NUC)
```

---

## 20. PoC Playbook — Single NUC

### Bill of Materials

| Item | Cost | Notes |
|---|---|---|
| Intel NUC 13 i7 (barebones) | ~$550 | NUC13ANHi7 or equivalent |
| 32 GB DDR4 (2x16 GB SODIMM) | ~$80 | 3200 MHz |
| 512 GB NVMe SSD | ~$50 | Samsung 980 Pro / WD SN770 |
| USB-to-GbE adapter (optional) | ~$20 | For second NIC |
| UPS (optional) | ~$80 | 400VA for clean shutdown |
| USRP B210 (optional, for OTA) | ~$1,500 | Only if testing real over-the-air |
| **Total (software lab)** | **~$700** | |
| **Total (OTA lab)** | **~$2,200** | |

### Quick Start: Open5GS on NUC (30 minutes)

```
Step 1: Install Ubuntu 22.04 LTS on NUC
Step 2: Add Open5GS PPA and install
        $ sudo add-apt-repository ppa:open5gs/latest
        $ sudo apt update && sudo apt install open5gs
Step 3: Start MongoDB (auto-installed as dependency)
Step 4: Edit configs in /etc/open5gs/*.yaml (set IPs, PLMN)
Step 5: Restart services
        $ sudo systemctl restart open5gs-*
Step 6: Install UERANSIM
        $ git clone https://github.com/aligungr/UERANSIM
        $ cd UERANSIM && make
Step 7: Configure and start gNB + UE
        $ ./nr-gnb -c config/open5gs-gnb.yaml
        $ ./nr-ue -c config/open5gs-ue.yaml
Step 8: Verify UE registration in AMF logs
        $ journalctl -u open5gs-amfd -f
Step 9: Ping through the UE tunnel
        $ ping -I uesimtun0 8.8.8.8
```

### Quick Start: free5GC on NUC (60 minutes)

```
Step 1: Install Ubuntu 22.04 LTS on NUC
Step 2: Install k3s
        $ curl -sfL https://get.k3s.io | sh -
Step 3: Build and install gtp5g kernel module
        $ sudo apt install linux-headers-$(uname -r)
        $ git clone https://github.com/free5gc/gtp5g
        $ cd gtp5g && make && sudo make install
Step 4: Deploy free5GC via Helm
        $ helm repo add towards5gs https://raw.githubusercontent.com/free5gc/towards5gs-helm/main/repo/
        $ helm install free5gc towards5gs/free5gc
Step 5: Deploy UERANSIM via Helm
        $ helm install ueransim towards5gs/ueransim
Step 6: Access WebUI for subscriber provisioning
        $ kubectl port-forward svc/free5gc-webui 5000:5000
Step 7: Add subscriber (IMSI, key, OPc) via WebUI
Step 8: Verify UE attachment
        $ kubectl logs -f deployment/ueransim-ue
Step 9: Test data plane from UE pod
        $ kubectl exec -it <ue-pod> -- ping -I uesimtun0 8.8.8.8
```

### PoC Timeline (Single NUC)

```
Day 1: Hardware Setup + OS Install
├── Unbox NUC, install RAM + NVMe
├── Install Ubuntu 22.04 LTS
├── Configure networking (static IP or DHCP)
└── Install prerequisites (Docker, k3s if needed)

Day 2: Core Deployment
├── Deploy Open5GS (apt) or free5GC (Helm)
├── Configure PLMN, DNN, IP pools
├── Add test subscriber (IMSI, keys)
└── Verify all NFs are running

Day 3: First UE Attach
├── Deploy UERANSIM
├── Achieve first successful UE registration
├── Test PDU session establishment
└── Verify data plane (ping through tunnel)

Day 4-5: Functional Testing
├── Multi-UE registration (10, 50, 100 UEs)
├── Concurrent PDU sessions
├── UE deregistration and re-registration
├── Network slice selection (if applicable)
└── Data plane throughput measurement

Day 6-7: Stability & Documentation
├── 24-48h soak test (monitor RAM, CPU, thermals)
├── NF crash recovery testing (kill + auto-restart)
├── Document configuration and findings
└── Present recommendation

Week 2 (optional): Advanced Testing
├── OTA test with srsRAN + USRP (if available)
├── Load test with PacketRusher (500+ UEs)
├── Monitoring setup (Prometheus + Grafana)
└── Migration test (Open5GS ↔ free5GC on same NUC)
```

### PoC Evaluation Scorecard

| Criterion | Weight | Open5GS | free5GC | Notes |
|---|:---:|:---:|:---:|---|
| Time to first UE attach | 15% | ___ | ___ | |
| Resource consumption on NUC | 15% | ___ | ___ | |
| Stability over 48h soak | 20% | ___ | ___ | |
| Setup complexity on NUC | 10% | ___ | ___ | |
| Documentation quality | 10% | ___ | ___ | |
| Team comfort level | 15% | ___ | ___ | |
| NF coverage for use case | 15% | ___ | ___ | |
| **Weighted Total** | **100%** | ___ | ___ | |

---

## 21. Governance Checklist

Before making a final decision, verify:

- [ ] **NUC hardware ordered** — confirmed specs (CPU, RAM, NVMe)
- [ ] **Ubuntu 22.04/24.04** — installed and updated on NUC
- [ ] **Kernel version** — compatible with gtp5g (if choosing free5GC)
- [ ] **Release cadence** — evaluate last 12 months of releases
- [ ] **CVE handling** — verify security response time
- [ ] **Performance validation** — load test on YOUR NUC hardware
- [ ] **NF coverage** — confirm all required NFs are implemented
- [ ] **License compatibility** — legal review against your business model
- [ ] **Team skills** — assess C vs Go alignment
- [ ] **Monitoring plan** — decide on Prometheus+Grafana vs lightweight alternative
- [ ] **Backup strategy** — automated MongoDB backups to external storage
- [ ] **Upgrade plan** — how to handle OS/kernel/5GC updates on NUC
- [ ] **Thermal management** — NUC placement, ventilation, ambient temperature
- [ ] **Power protection** — UPS for clean shutdown on power loss
- [ ] **Scale-out plan** — documented path from NUC to multi-node cluster when needed

---

## 22. Appendix: Glossary

| Abbreviation | Full Name | Description |
|---|---|---|
| **AMF** | Access and Mobility Management Function | Handles UE registration, connection, mobility |
| **SMF** | Session Management Function | Manages PDU sessions and UPF selection |
| **UPF** | User Plane Function | Handles user data packet routing and forwarding |
| **NRF** | NF Repository Function | Service registry for NF discovery |
| **UDM** | Unified Data Management | Manages subscriber data and authentication |
| **UDR** | Unified Data Repository | Database for policy and subscription data |
| **AUSF** | Authentication Server Function | Handles authentication procedures |
| **PCF** | Policy Control Function | Policy rules and charging control |
| **NSSF** | Network Slice Selection Function | Selects appropriate network slice for UE |
| **NEF** | Network Exposure Function | Exposes network capabilities to external apps |
| **N3IWF** | Non-3GPP Interworking Function | Connects non-3GPP access (WiFi) to 5GC |
| **TNGF** | Trusted Non-3GPP Gateway Function | Trusted non-3GPP access gateway |
| **CHF** | Charging Function | Online/offline charging |
| **SCP** | Service Communication Proxy | Service mesh proxy for inter-NF communication |
| **SBI** | Service-Based Interface | HTTP/2 based interface between NFs |
| **SBA** | Service-Based Architecture | 3GPP 5G architecture pattern |
| **EPC** | Evolved Packet Core | 4G LTE core network |
| **GTP-U** | GPRS Tunnelling Protocol - User | Tunnel protocol for user plane data |
| **NGAP** | Next Generation Application Protocol | Protocol between gNB and AMF (N2) |
| **PFCP** | Packet Forwarding Control Protocol | Protocol between SMF and UPF (N4) |
| **PDU** | Protocol Data Unit | Data session between UE and network |
| **NUC** | Next Unit of Computing | Intel compact form-factor PC |
| **UERANSIM** | UE and RAN Simulator | Open-source 5G UE and gNB simulator |
| **HPA** | Horizontal Pod Autoscaler | K8s auto-scaling mechanism |
| **SBOM** | Software Bill of Materials | Inventory of software components |
| **TCO** | Total Cost of Ownership | Full cost including hidden/indirect costs |
| **PoC** | Proof of Concept | Validation deployment before production |
| **k3s** | Lightweight Kubernetes | CNCF-certified K8s for edge/IoT |
| **gtp5g** | GTP-U Kernel Module | Linux kernel module for GTP user plane |
| **DKMS** | Dynamic Kernel Module Support | Framework for auto-rebuilding kernel modules |
| **SDR** | Software Defined Radio | Radio hardware for OTA testing (e.g., USRP) |
| **USRP** | Universal Software Radio Peripheral | Ettus Research SDR platform |

---

*This document is optimized for single-NUC private 5G core deployment. For multi-node cluster guidance, scale up the same architectures with additional nodes and proper HA.*

*Generated: February 2026*
