# 5G Core Network Comparison: Open5GS vs free5GC vs Radisys

> **Document Type:** Technical Evaluation & Decision Guide
> **Scope:** Single-NUC Lab / PoC / Private 5G — **Non-Commercial Grade**
> **Audience:** Platform Engineering, CTO, Network Architects
> **Hardware Target:** Intel NUC 15 Pro (Arrow Lake, 2025)
> **Scale Target:** ~20 attached UEs, ~200 UE identification, ~400 UE reject
> **Last Updated:** February 2026
> **Status:** ACTIVE — Review quarterly

---

## Table of Contents

1. [Executive Summary & Verdict](#1-executive-summary--verdict)
2. [At a Glance — Scorecard](#2-at-a-glance--scorecard)
3. [NUC 15 Pro — Target Hardware](#3-nuc-15-pro--target-hardware)
4. [Scale Profile — What We're Actually Testing](#4-scale-profile--what-were-actually-testing)
5. [Architecture Deep Dive](#5-architecture-deep-dive)
6. [Network Function Coverage Matrix](#6-network-function-coverage-matrix)
7. [Language, Code Quality & Developer Experience](#7-language-code-quality--developer-experience)
8. [Resource Footprint on NUC 15](#8-resource-footprint-on-nuc-15)
9. [3GPP Standards Compliance](#9-3gpp-standards-compliance)
10. [Open Source Health & Governance](#10-open-source-health--governance)
11. [Security & Supply Chain](#11-security--supply-chain)
12. [Licensing Deep Dive](#12-licensing-deep-dive)
13. [Day-2 Operations](#13-day-2-operations)
14. [Resilience on a Single Node](#14-resilience-on-a-single-node)
15. [Testing & Simulation Ecosystem](#15-testing--simulation-ecosystem)
16. [RAN Interoperability](#16-ran-interoperability)
17. [Migration Paths](#17-migration-paths)
18. [Total Cost of Ownership — NUC 15 Edition](#18-total-cost-of-ownership--nuc-15-edition)
19. [Risk Register](#19-risk-register)
20. [Decision Framework](#20-decision-framework)
21. [PoC Playbook — NUC 15](#21-poc-playbook--nuc-15)
22. [Governance Checklist](#22-governance-checklist)
23. [Appendix: Glossary](#23-appendix-glossary)

---

## 1. Executive Summary & Verdict

### Context

This is **not** a commercial-grade deployment evaluation. We're building a **lab/PoC private 5G core** on a single Intel NUC 15 Pro for:
- **~20 simultaneously attached UEs** (active PDU sessions)
- **~200 UE identifications** (registration attempts, auth cycles)
- **~400 UE rejects** (testing reject scenarios, overload behavior)

This is a development/test/demo workload — not carrier-grade.

### The Short Version

| | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **One-liner** | Lean C core — full 5GC + EPC in ~2 GB RAM | Go core — **all CP in a single Docker image**, lightweight for lab | Vendor black-box — overkill for lab use |
| **Best for** | Smallest footprint, hybrid LTE+5G, bare-metal simplicity | Single-container lab, K8s-optional, Apache 2.0 license | Zero in-house expertise, unlimited budget |
| **NUC 15 fit** | **Excellent** — barely touches the hardware | **Excellent** — single container makes it trivial | **Poor** — not designed for single-node |
| **License** | AGPLv3 (copyleft) | Apache 2.0 (permissive) | Proprietary |
| **Language** | C | Go | Undisclosed |
| **Deployment for lab** | `apt install` or Docker Compose | **Single Docker image** (all CP NFs) + UPF | Vendor professional services |
| **Year-1 NUC TCO** | **$800-1.5K** | **$800-1.5K** | **$200K+** (absurd for lab) |

### Decision Matrix for Our Scale

| If your priority is... | Recommendation |
|---|---|
| Absolute simplest setup on NUC 15 | **Open5GS** (`apt install`) |
| Single Docker image for entire control plane | **free5GC** (all-in-one image) |
| Hybrid LTE + 5G on same NUC | **Open5GS** (only option) |
| Apache 2.0 license | **free5GC** |
| Broadest NF coverage (N3IWF, CHF, NEF) | **free5GC** |
| No kernel module hassle | **Open5GS** |
| Testing 20 attached + 200 ident + 400 reject | **Either** — both handle this effortlessly |
| Vendor SLA, turnkey | **Radisys** (but not on a NUC, not for lab) |

---

## 2. At a Glance — Scorecard

### Criteria Ratings (1-5, higher is better)

| Criteria | Open5GS | free5GC | Radisys | Notes |
|---|:---:|:---:|:---:|---|
| **NUC 15 Deployability** | **5** | **5** | 1 | Both trivial on NUC 15; free5GC single-image changes the game |
| **Deployment Simplicity** | **5** | 4 | 2 | Open5GS: apt install. free5GC: single docker run + UPF |
| **Cloud-Native Readiness** | 2 | **5** | 3 | free5GC still K8s-ready if needed later |
| **Resource Efficiency** | **5** | 4 | 2 | Both are light at 20-UE scale; Open5GS still leaner |
| **NF Coverage Breadth** | 3 | **5** | 3 | free5GC: N3IWF, TNGF, CHF, NEF |
| **4G/5G Hybrid Support** | **5** | 1 | 2 | Open5GS: EPC + 5GC in one codebase |
| **Code Auditability** | 5 | 5 | 0 | OSS = full audit; Radisys = black box |
| **Customizability** | 5 | 5 | 1 | Source access vs vendor request queue |
| **Vendor Support / SLA** | 1 | 1 | 5 | Only Radisys offers contractual SLA |
| **Hiring Pool** | 2 | 4 | N/A | Go devs >> C + telecom specialists |
| **License Flexibility** | 2 | **5** | 1 | Apache 2.0 is most business-friendly |
| **Supply Chain Control** | 5 | 5 | 0 | Full SBOM, image signing for OSS |
| **Community & Ecosystem** | 4 | 4 | 1 | Both OSS have active communities |
| **Lab/PoC Suitability** | **5** | **5** | 1 | Both are made for this; Radisys is not |
| **Total (out of 70)** | **54** | **58** | **22** | free5GC edges ahead for lab with single-image simplicity |

> **At our scale (20 attached / 200 ident / 400 reject):** Both OSS options are **overkill** — the NUC 15 will be mostly idle. The choice comes down to deployment style preference (apt vs Docker) and license needs.

---

## 3. NUC 15 Pro — Target Hardware

### Intel NUC 15 Pro (Arrow Lake, 2025)

The NUC 15 Pro is Intel's latest compact PC platform, built on Arrow Lake architecture.

| Spec | NUC 15 Pro (Typical Config) |
|---|---|
| **CPU** | Intel Core Ultra (Arrow Lake) — 12-16 cores, up to 4.8+ GHz |
| **Architecture** | Arrow Lake — hybrid P-core + E-core |
| **RAM** | DDR5 — 32 GB or 64 GB (2x SODIMM slots) |
| **Storage** | M.2 NVMe Gen4 — 512 GB to 2 TB |
| **Networking** | 2.5GbE Intel i226-V, WiFi 7, Bluetooth 5.4 |
| **USB** | Thunderbolt 4 / USB4, USB 3.2 Gen 2, USB-A |
| **TDP** | 15-28W (configurable) |
| **Form Factor** | ~117 x 112 x 37 mm |
| **OS Support** | Ubuntu 22.04/24.04, Windows 11 |

### Why NUC 15 Pro is Generous for Our Use Case

```
┌─────────────────────────────────────────────────────────────────┐
│  Intel NUC 15 Pro — Resource Budget for 5GC Lab                  │
│                                                                  │
│  Available:  12-16 cores │ 32-64 GB DDR5 │ 512 GB+ NVMe         │
│                                                                  │
│  5GC needs:  0.5-2 cores │ 2-4 GB RAM    │ <1 GB disk           │
│  (at our 20-UE scale)                                            │
│                                                                  │
│  ┌──────────────────────────────────────────────────────┐        │
│  │  Used by 5GC          │  Free for everything else    │        │
│  │  ██░░░░░░░░░░░░░░░░░  │  Development, monitoring,   │        │
│  │  ~10-15% of NUC 15    │  other services, browsing   │        │
│  └──────────────────────────────────────────────────────┘        │
│                                                                  │
│  The NUC 15 is MASSIVELY overpowered for 20 attached UEs.       │
│  This is good — it means zero resource concerns.                 │
└─────────────────────────────────────────────────────────────────┘
```

### NUC 15 Networking

The NUC 15 Pro has a **single 2.5GbE** port. For a 20-UE lab, this is a non-issue.

```
┌─────────────────────────────────────────────────────────┐
│  NUC 15 Networking for Lab                               │
│                                                         │
│  Option 1: All Virtual (UERANSIM on same NUC)           │
│  ┌──────────┐                                           │
│  │ eth0     │──── Internet/Management                   │
│  │ 2.5GbE   │                                          │
│  │          │  Docker bridge / Linux bridges for        │
│  │          │  all N2/N3/N6/SBI traffic internally     │
│  └──────────┘                                           │
│  (Simplest — everything is localhost)                   │
│                                                         │
│  Option 2: USB-C to GbE (external gNB/RAN)             │
│  ┌──────────┐  ┌──────────────────┐                    │
│  │ eth0     │  │ eth1 (USB-C)     │                    │
│  │ 2.5GbE   │  │ Thunderbolt dock │                    │
│  │ Internet │  │ RAN traffic      │                    │
│  └──────────┘  └──────────────────┘                    │
│                                                         │
│  For 20 UEs, 2.5GbE is 100x more than needed.          │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Scale Profile — What We're Actually Testing

### Non-Commercial Lab Workload

This section defines what we **actually** need the 5GC to handle. We are **not** building carrier-grade infrastructure.

| Metric | Target | Context |
|---|---|---|
| **Simultaneously attached UEs** | ~20 | Active PDU sessions, data flowing |
| **UE identifications** | ~200 | Registration + authentication cycles |
| **UE rejects** | ~400 | Testing reject scenarios (auth fail, overload, slice mismatch) |
| **Peak registrations/sec** | 5-10 | Not a stress test — functional validation |
| **Data plane throughput** | 10-50 Mbps aggregate | Enough to verify tunneling works |
| **PDU sessions** | 20-30 concurrent | One per attached UE, maybe a few multi-PDU |
| **Network slices** | 1-3 | Basic slice selection testing |

### What This Means for Resource Planning

```
┌────────────────────────────────────────────────────────────────┐
│  Scale Reality Check                                            │
│                                                                │
│  20 attached UEs is NOTHING for modern 5GC implementations.    │
│  Both Open5GS and free5GC can handle 500-1000+ UEs             │
│  on modest hardware.                                            │
│                                                                │
│  At 20 UEs:                                                     │
│  ├── AMF: essentially idle (0.01 core, 30-50 MB)               │
│  ├── SMF: essentially idle (0.01 core, 30-50 MB)               │
│  ├── UPF: barely working (0.05 core, 50-100 MB)               │
│  ├── Other NFs: sleeping (0.01 core each)                      │
│  └── Total 5GC: ~0.2 cores, ~500 MB-1 GB RAM                  │
│                                                                │
│  The NUC 15 Pro won't even notice the 5GC is running.          │
│                                                                │
│  200 ident + 400 reject happen in burst scenarios:              │
│  ├── Peak: maybe 1-2 cores for a few seconds                   │
│  ├── AMF handles all of these — it's just NAS signaling        │
│  └── MongoDB writes are tiny (~1 KB per UE context)            │
└────────────────────────────────────────────────────────────────┘
```

### Resource at Our Scale (Measured Estimates)

| Component | CPU at 20 UEs | RAM at 20 UEs | Notes |
|---|---|---|---|
| **Entire 5GC (all NFs)** | 0.1-0.3 core | 500 MB - 1 GB | Both Open5GS and free5GC |
| **MongoDB** | 0.05 core | 200-500 MB | Cap WiredTiger cache at 256 MB for lab |
| **UERANSIM (20 UEs + gNB)** | 0.1 core | 50-100 MB | Lightweight at this scale |
| **Docker / containerd** | 0.05 core | 100-200 MB | If using containers |
| **Total workload** | **0.3-0.5 core** | **1-2 GB** | Out of 12-16 cores and 32-64 GB |
| **NUC 15 utilization** | **~3%** | **~3-6%** | Massively over-provisioned |

> **Key insight:** At this scale, the choice between Open5GS and free5GC is **not about performance or resources** — both are trivial on a NUC 15. The choice is about **deployment style, license, and feature needs**.

---

## 5. Architecture Deep Dive

### 5.1 Open5GS

```
┌──────────────────────────────────────────────────────────────┐
│  Intel NUC 15 Pro (Arrow Lake, 32+ GB DDR5)                   │
│  Ubuntu 22.04/24.04 LTS                                       │
│                                                               │
│  ┌── Open5GS (all-in-one, systemd services) ──────────────┐  │
│  │                                                         │  │
│  │  5G Core (SBA)              EPC (4G)                    │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐  ┌─────┐ ┌─────┐            │  │
│  │  │ AMF │ │ SMF │ │ UPF │  │ MME │ │ HSS │            │  │
│  │  └─────┘ └─────┘ └─────┘  └─────┘ └─────┘            │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐  ┌─────┐ ┌─────┐            │  │
│  │  │ NRF │ │ UDM │ │ UDR │  │ SGW │ │ PGW │            │  │
│  │  └─────┘ └─────┘ └─────┘  └─────┘ └─────┘            │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌──────┐            │  │
│  │  │AUSF │ │ PCF │ │NSSF │ │ SCP │ │ PCRF │            │  │
│  │  └─────┘ └─────┘ └─────┘ └─────┘ └──────┘            │  │
│  │                                                         │  │
│  │  At 20 UEs: ~500 MB RAM, ~0.2 cores                    │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌── MongoDB ──────┐  ┌── UERANSIM ─────────────────────┐    │
│  │  ~200 MB RAM     │  │  gNB + 20 UEs: ~50 MB RAM       │    │
│  │  (lab-sized)     │  │  ~0.1 core                       │    │
│  └──────────────────┘  └──────────────────────────────────┘    │
│                                                               │
│  Total: ~1-2 GB used  │  CPU: ~0.3-0.5 cores                 │
│  NUC 15 barely notices this workload                          │
└──────────────────────────────────────────────────────────────┘
```

**Architecture Philosophy:** Monolithic-leaning, spec-faithful. Each NF is a lightweight C binary (single process). Uniquely supports **both EPC (4G) and 5GC** in a single codebase.

| Property | Detail |
|---|---|
| **NF Design** | Single-process per NF, lightweight C binaries |
| **Inter-NF Communication** | SBI (HTTP/2) for 5GC; GTP-C/GTP-U for EPC |
| **Data Store** | MongoDB (shared) |
| **Configuration** | YAML files per NF (`/etc/open5gs/*.yaml`) |
| **Build System** | Meson + Ninja |
| **Deployment** | `apt install open5gs` — that's it |
| **UPF Implementation** | Userspace (libgtpnl + TUN) — **no kernel module** |
| **Container Count** | 0 (bare metal) or 1 Docker Compose if preferred |
| **RAM at 20 UEs** | ~500 MB for all NFs combined |

**NUC 15 Strengths:**
- `apt install open5gs` on Ubuntu — running in 5 minutes
- ~500 MB RAM at our scale — NUC 15 has 32-64 GB
- No kernel module — works on any stock Ubuntu kernel
- Hybrid 4G/5G — test both EPC and 5GC without separate stacks
- systemd manages everything — familiar Linux operations
- Individual NF processes = easy to restart one without touching others

**Weaknesses:**
- No built-in WebUI (use MongoDB CLI or community tools)
- AGPLv3 — copyleft if you expose as a service
- C codebase — harder to contribute to without telecom + C skills
- Bus factor — Sukchan Lee is the primary maintainer

---

### 5.2 free5GC — Single Docker Image Deployment

**Key change from previous analysis:** free5GC's entire control plane can run in a **single Docker image**. You don't need 10-14 separate pods. For a lab with 20 UEs, one container runs all CP NFs.

```
┌──────────────────────────────────────────────────────────────┐
│  Intel NUC 15 Pro (Arrow Lake, 32+ GB DDR5)                   │
│  Ubuntu 22.04/24.04 LTS + Docker                              │
│                                                               │
│  ┌── free5GC (Single Docker Container — ALL CP NFs) ───────┐  │
│  │                                                          │  │
│  │  ┌───────────────────────────────────────────────────┐   │  │
│  │  │  One Container, All Control Plane:                │   │  │
│  │  │                                                   │   │  │
│  │  │  AMF │ SMF │ NRF │ UDM │ UDR │ AUSF │ PCF │     │   │  │
│  │  │  NSSF │ N3IWF │ CHF │ NEF │ WebUI                │   │  │
│  │  │                                                   │   │  │
│  │  │  All NFs share one process space                  │   │  │
│  │  │  SBI communication via localhost                  │   │  │
│  │  └───────────────────────────────────────────────────┘   │  │
│  │                                                          │  │
│  │  At 20 UEs: ~500 MB - 1 GB RAM, ~0.2 cores              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌── UPF (separate container or host process) ─────────────┐  │
│  │  Requires gtp5g kernel module                            │  │
│  │  ~100-200 MB RAM at 20 UEs                               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌── MongoDB ──────┐  ┌── UERANSIM ─────────────────────┐    │
│  │  ~200 MB RAM     │  │  gNB + 20 UEs: ~50 MB RAM       │    │
│  │  (lab-sized)     │  │  ~0.1 core                       │    │
│  └──────────────────┘  └──────────────────────────────────┘    │
│                                                               │
│  Total: ~1-2 GB used  │  CPU: ~0.3-0.5 cores                 │
│  3-4 containers total (not 10-14!)                            │
└──────────────────────────────────────────────────────────────┘
```

**The Single Docker Image Advantage:**

Previous evaluations assumed free5GC needed 10-14 separate pods/containers. In practice, for lab use:

| Deployment Style | Containers | Complexity | Best For |
|---|---|---|---|
| **Single CP image + UPF** | 3-4 (CP + UPF + MongoDB + UERANSIM) | **Low** | Lab, PoC, demo |
| **Per-NF containers (Helm)** | 10-14 pods + K8s | High | Production, scaling |
| **Docker Compose (per-NF)** | 10-14 containers | Medium | Dev, integration testing |

For our 20-UE lab, the **single CP image** approach is the clear winner — it eliminates K8s overhead, inter-container networking complexity, and resource duplication.

| Property | Detail |
|---|---|
| **NF Design** | All CP NFs bundled in one Docker image |
| **Inter-NF Communication** | SBI via localhost (all in same container) |
| **Data Store** | MongoDB (separate container) |
| **Configuration** | YAML files mounted into container |
| **Deployment** | `docker run` for CP + `docker run` for UPF |
| **UPF** | Separate container (needs gtp5g kernel module + host networking) |
| **Container Count** | **3-4 total** (CP + UPF + MongoDB + UERANSIM) |
| **RAM at 20 UEs** | ~500 MB-1 GB for CP container |
| **WebUI** | Built-in, exposed on port 5000 |

**NUC 15 Strengths:**
- **Single Docker image** = trivial deployment, easy to start/stop/upgrade
- Built-in WebUI for subscriber provisioning (no MongoDB CLI needed)
- Apache 2.0 license — no copyleft concerns
- Broadest NF coverage: N3IWF, TNGF, CHF, NEF all included
- Go language — memory-safe, modern tooling
- Can scale to K8s later without changing the core — just split into per-NF pods
- Docker image is portable — same image on NUC, laptop, cloud

**Weaknesses:**
- gtp5g kernel module required for UPF (must compile on NUC 15 kernel)
- Slightly larger RAM footprint than Open5GS (Go runtime overhead)
- No 4G EPC support — 5G SA only
- UPF container needs `--privileged` and `--network host`

---

### 5.3 Radisys 5G Core

```
┌──────────────────────────────────────────────────────┐
│              Radisys 5G Core (Vendor)                 │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │       Pre-built NF Bundle (Opaque)             │  │
│  │  AMF | SMF | UPF | NRF | UDM | AUSF | PCF     │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  Management & Orchestration Portal             │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  Designed for: multi-node server clusters             │
│  Not designed for: single NUC, lab use                │
│  Cost: $200K+ — absurd for 20-UE lab                 │
└──────────────────────────────────────────────────────┘
```

**Lab Reality Check:** Radisys is a commercial-grade vendor solution designed for carrier deployments. Using it for a 20-UE lab on a NUC 15 would be like renting a Boeing 747 to fly across a parking lot. Included here for completeness only.

---

## 6. Network Function Coverage Matrix

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

**For our 20-UE lab:**
- Core NFs (AMF, SMF, UPF, NRF, UDM, UDR, AUSF): both have them all
- If testing WiFi offload (N3IWF) or charging (CHF): free5GC only
- If testing 4G LTE + 5G hybrid: Open5GS only
- If just testing basic 5G SA registration, PDU sessions, and data plane: either works

---

## 7. Language, Code Quality & Developer Experience

### Side-by-Side Comparison

| Aspect | Open5GS (C) | free5GC (Go) | Radisys |
|---|---|---|---|
| **Memory Safety** | Manual — careful coding needed | GC-managed — safe by default | Cannot audit |
| **Performance** | Excellent — deterministic latency | Good — slight GC overhead | Unverifiable |
| **Concurrency** | Event-driven (epoll/kqueue) | Goroutines — lightweight threads | Unknown |
| **Testability** | Unit tests; fewer C frameworks | Go built-in `testing` package | Internal QA only |
| **Build** | Meson + Ninja (~2 min) | `go build` (~1 min per NF) | N/A |
| **Debugging** | GDB, Valgrind, AddressSanitizer | Delve, pprof — modern, accessible | Vendor tickets |
| **IDE Support** | Good (CLion, VS Code + C) | Excellent (GoLand, VS Code + Go) | N/A |
| **Hiring Difficulty** | **High** — C + telecom | **Low-Medium** — large Go pool | N/A |
| **Onboarding Time** | 4-8 weeks | 2-4 weeks | N/A |

### Developer Workflow on NUC 15

| Workflow | Open5GS | free5GC |
|---|---|---|
| **Install** | `apt install open5gs` (5 min) | `docker pull` + `docker run` (5 min) |
| **Add subscriber** | `mongosh` CLI command | WebUI at `localhost:5000` |
| **Restart one NF** | `systemctl restart open5gs-amfd` | Restart container (or just restart all — it's one container) |
| **View logs** | `journalctl -u open5gs-amfd -f` | `docker logs -f free5gc-cp` |
| **Modify config** | Edit `/etc/open5gs/amf.yaml`, restart service | Edit mounted YAML, restart container |
| **Debug a crash** | GDB + core dump | Delve + goroutine traces |

---

## 8. Resource Footprint on NUC 15

### At Our Scale (20 Attached UEs)

| Component | Open5GS (bare metal) | free5GC (single Docker) | Notes |
|---|---|---|---|
| **All CP NFs** | ~300-500 MB | ~500 MB - 1 GB | Go runtime adds ~200-400 MB overhead |
| **UPF** | ~50-100 MB | ~100-200 MB | gtp5g kernel module for free5GC |
| **MongoDB** | ~200 MB | ~200 MB | Cap WiredTiger cache at 256 MB |
| **UERANSIM** | ~50 MB | ~50 MB | 20 UEs is trivial |
| **Docker overhead** | 0 (bare metal) | ~100 MB | containerd + Docker daemon |
| **Total** | **~600 MB - 1 GB** | **~1-1.5 GB** | |
| **CPU (steady)** | ~0.2 core | ~0.2 core | |
| **CPU (peak burst)** | ~1-2 cores (seconds) | ~1-2 cores (seconds) | During 200-ident / 400-reject burst |
| **NUC 15 RAM used** | ~2-3% of 32 GB | ~3-5% of 32 GB | Both are negligible |
| **NUC 15 CPU used** | ~1-2% of 12 cores | ~1-2% of 12 cores | Both are negligible |

### Per-NF Breakdown (at 20 UEs)

| NF | CPU | RAM | Traffic at 20 UEs |
|---|---|---|---|
| **AMF** | 0.01 core | 30-50 MB | Handles all 200 ident + 400 reject |
| **SMF** | 0.01 core | 30-50 MB | 20 PDU session setups |
| **UPF** | 0.05 core | 50-100 MB | 10-50 Mbps aggregate user data |
| **NRF** | 0.005 core | 20-30 MB | Initial NF registration, then idle |
| **UDM** | 0.005 core | 20-30 MB | Auth data lookups |
| **UDR** | 0.005 core | 20-30 MB | Subscriber data reads |
| **AUSF** | 0.005 core | 20-30 MB | Auth challenges |
| **PCF** | 0.005 core | 20-30 MB | Policy lookups |
| **NSSF** | 0.002 core | 15-20 MB | Slice selection |

### NUC 15 Has Massive Headroom

```
NUC 15 Pro Resource Utilization at 20 UEs

RAM (32 GB):
██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  5GC (~1-2 GB)
░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  Free (30+ GB)

CPU (12 cores):
█░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  5GC (~0.3 core)
░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  Free (11.7+ cores)

Disk (512 GB):
█░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  5GC (<1 GB)
░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  Free (511+ GB)

You could run 10-20 instances of the entire 5GC on this NUC.
```

### Thermal on NUC 15

At our 20-UE workload, thermal is a **non-issue**:

| Scenario | NUC 15 Temp | Fan | Sustainable? |
|---|---|---|---|
| 5GC idle + 20 UEs attached | 35-45°C | Silent | Indefinitely |
| 200 ident + 400 reject burst | 40-50°C | Barely audible | Indefinitely |
| Stress test (500+ UEs) | 55-70°C | Noticeable | Yes |

---

## 9. 3GPP Standards Compliance

| Aspect | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Target 3GPP Release** | Release 17 (partial Rel-18) | Release 15/16 (some Rel-17) | Vendor-claimed Rel-16+ |
| **Spec Traceability** | Direct C → TS spec mapping | Go SBA → TS spec mapping | Opaque |
| **SBI Compliance** | HTTP/2 based SBI | HTTP/2 based SBI | Vendor implementation |
| **NAS Protocol** | Full 5G NAS | Full 5G NAS | Vendor |
| **NGAP (N2)** | Full | Full | Vendor |
| **PFCP (N4)** | Full | Full | Vendor |
| **GTP-U (N3/N9)** | Userspace (portable) | Kernel module (faster) | Vendor |
| **Network Slicing** | NSSF (basic) | NSSF + NEF | Vendor claim |
| **Non-3GPP Access** | Not yet | N3IWF + TNGF | Vendor claim |
| **Interop Tested** | UERANSIM, srsRAN, commercial gNBs | UERANSIM, srsRAN, commercial gNBs | Vendor lab |

**For our 20-UE lab:** Both are 3GPP-compliant enough for registration, authentication, PDU session establishment, and data plane testing. The standards differences matter more at commercial scale.

---

## 10. Open Source Health & Governance

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
| **Documentation** | open5gs.org | free5gc.org |
| **Community** | Discord, mailing list | Discord, forums |

### Bus Factor

| | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Key Person Risk** | **HIGH** — Sukchan Lee | **MEDIUM** — NYCU team | **LOW** — vendor org |
| **Mitigation** | Forkable (AGPLv3); growing contributors | Forkable (Apache 2.0) | SLA; vendor bankruptcy = risk |

---

## 11. Security & Supply Chain

| Control | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Build from source** | Yes | Yes | No |
| **Reproducible builds** | Yes (Meson) | Yes (Go modules) | No |
| **SBOM generation** | Full (Syft, Trivy) | Full (Syft, Trivy) | Limited/None |
| **CVE scanning** | Full (Trivy, Grype) | Full (Trivy, Grype) | Vendor responsibility |
| **Patch turnaround** | Self-controlled (hours) | Self-controlled (hours) | Vendor SLA (days/weeks) |
| **Full source audit** | Yes | Yes | **No** |
| **TLS/mTLS config** | Configurable per NF | Configurable per NF | Vendor-managed |

### Lab Security Note

On a single NUC 15 for lab use, security is less critical than production. That said:
- All NFs share the same host — localhost SBI means no network-level separation
- free5GC UPF runs as privileged — acceptable for lab
- Use a firewall or isolated VLAN if the NUC is on a shared network
- Don't expose the NUC's 5GC ports to the internet (obviously)

```
Supply Chain Control

  FULL CONTROL ◄───────────────────────────► ZERO CONTROL
  ┌──────────┐  ┌──────────┐      ┌──────────┐
  │ Open5GS  │  │ free5GC  │      │ Radisys  │
  │ Build    │  │ Build    │      │ Receive  │
  │ from src │  │ from src │      │ opaque   │
  │ Audit    │  │ or pull  │      │ images   │
  │ Sign     │  │ Docker   │      │ Trust    │
  └──────────┘  └──────────┘      └──────────┘
```

---

## 12. Licensing Deep Dive

| Scenario | Open5GS (AGPLv3) | free5GC (Apache 2.0) | Radisys (Proprietary) |
|---|---|---|---|
| **Internal lab on NUC 15** | Safe — no obligations | Safe — no obligations | Per contract |
| **Modifications for internal use** | Safe — internal use | Safe — no obligations | Per contract |
| **Offering as a network service** | **Must share modified source** (AGPL) | No obligation | N/A |
| **Embedding in commercial product** | **Copyleft — must open-source derivative** | Permissive — just attribution | Per contract |
| **Forking for proprietary use** | **Not allowed** | **Allowed** | N/A |

> **For our lab:** Both licenses are fine. If there's any chance of future commercialization, **free5GC's Apache 2.0** is safer.

---

## 13. Day-2 Operations

### Deployment & Management on NUC 15

| Aspect | Open5GS | free5GC (single Docker) |
|---|---|---|
| **Install** | `apt install open5gs` | `docker pull` + `docker-compose up` |
| **Config location** | `/etc/open5gs/*.yaml` | Mounted volume YAML files |
| **Add subscriber** | `mongosh` or community WebUI | Built-in WebUI on port 5000 |
| **Start/Stop** | `systemctl start/stop open5gs-*` | `docker-compose up/down` |
| **Logs** | `journalctl -u open5gs-amfd` | `docker logs free5gc-cp` |
| **Upgrade** | `apt upgrade open5gs` | `docker pull` new image, recreate |
| **Rollback** | `apt install open5gs=<version>` | `docker run` with old image tag |
| **Backup** | `mongodump` | `mongodump` |
| **Monitoring (lab)** | `journalctl` + `top` is enough | `docker stats` + `docker logs` is enough |

### Do You Need Prometheus/Grafana?

**No.** For 20 UEs, `journalctl`, `docker logs`, and `top`/`htop` are more than sufficient. Save the monitoring stack for when you scale beyond 100 UEs or move to a multi-node setup.

---

## 14. Resilience on a Single Node

For a **lab/PoC**, high availability is not a real requirement. But basic resilience helps avoid annoying restarts:

| Aspect | Open5GS | free5GC (Docker) |
|---|---|---|
| **Auto-restart on crash** | systemd `Restart=always` | Docker `restart: always` |
| **DB backup** | Daily `mongodump` to USB or NFS | Same |
| **Power loss** | UPS ($50-80) — nice to have | Same |
| **Full NUC failure** | Keep config/YAMLs in git — redeploy in 30 min | Keep docker-compose.yaml + YAMLs in git |

### Lab Resilience Checklist

- [ ] Auto-restart enabled for all NFs / containers
- [ ] Configuration files backed up (git repo)
- [ ] MongoDB backup script (weekly is fine for lab)
- [ ] Pin kernel version if using free5GC (avoid gtp5g breakage)

---

## 15. Testing & Simulation Ecosystem

| Tool / Simulator | Description | Open5GS | free5GC |
|---|---|:---:|:---:|
| **UERANSIM** | Open-source 5G UE + gNB simulator | Tested | Tested |
| **srsRAN** | Open-source 4G/5G RAN stack | Tested | Tested |
| **PacketRusher** | Go-based 5G UE/gNB load tester | Compatible | Compatible |
| **gNBSim** | ONF gNB simulator | Compatible | Compatible |
| **Commercial UEs** | Quectel, Sierra Wireless | Community tested | Community tested |
| **Commercial gNBs** | Baicells, Airspan | Community reports | Community reports |

### Test Plan for Our Scale

| Test | UEs | Expected Outcome | Tool |
|---|---|---|---|
| **Basic attach** | 1 UE | Registration + PDU session + ping | UERANSIM |
| **Multi-UE attach** | 20 UEs | All register, all get IP, all can ping | UERANSIM |
| **Identification burst** | 200 UEs | AMF handles 200 registrations | UERANSIM / PacketRusher |
| **Reject scenarios** | 400 UEs | Auth failures, slice mismatch, overload reject | UERANSIM with bad keys/PLMN |
| **Data plane** | 20 UEs | iperf through tunnel, measure throughput | UERANSIM + iperf3 |
| **Deregister/re-register** | 20 UEs | Clean detach, re-attach cycle | UERANSIM |
| **Soak test** | 20 UEs | 24-48h stable, no memory leaks | UERANSIM |

---

## 16. RAN Interoperability

| RAN Solution | Type | Open5GS | free5GC |
|---|---|:---:|:---:|
| **UERANSIM** | Simulator | Tested | Tested |
| **srsRAN** | Open-source | Tested | Tested |
| **Baicells** | Commercial small cell | Community tested | Community tested |
| **Airspan** | Commercial | Community reports | Community reports |

> For our lab, UERANSIM is the go-to. If testing OTA (over-the-air), add srsRAN + USRP B210.

---

## 17. Migration Paths

| Migration Path | Difficulty | Effort | Notes |
|---|:---:|---|---|
| **Open5GS → free5GC** (same NUC) | Low-Med | 1-2 days | Both use MongoDB — subscriber data portable. Install gtp5g. |
| **free5GC → Open5GS** (same NUC) | Low | 1 day | Remove gtp5g. Free up some RAM. Lose WebUI/N3IWF. |
| **Single Docker → K8s (free5GC)** | Low | 1 day | Split single image into per-NF Helm chart. Same configs. |
| **NUC → Multi-node cluster** | Low-Med | 1-2 weeks | Same Helm charts, more nodes. Add real HA. |
| **Either OSS → Radisys** | High | 4-8 weeks | Complete re-provisioning. Leave NUC world. |

> **Strategy:** Start with whichever feels right. At 20 UEs, migrating between Open5GS and free5GC is a 1-2 day exercise. Don't overthink it.

---

## 18. Total Cost of Ownership — NUC 15 Edition

### Hardware Cost

| Item | Cost | Notes |
|---|---|---|
| Intel NUC 15 Pro (barebones) | ~$600-800 | Arrow Lake, latest generation |
| 32 GB DDR5 (2x16 GB SODIMM) | ~$80-100 | More than enough |
| 512 GB NVMe Gen4 | ~$50 | Samsung 990 EVO or similar |
| **Total NUC 15** | **~$750-950** | Ready for 5GC lab |

### Year-1 Total Cost

| Component | Open5GS on NUC 15 | free5GC on NUC 15 | Radisys |
|---|---|---|---|
| **NUC 15 Hardware** | $800 | $800 | N/A |
| **Software License** | $0 | $0 | $100K-500K+/yr |
| **Setup Time** | 30 min ($0 — you do it) | 30 min ($0 — you do it) | Weeks + vendor PS |
| **Electricity** | ~$30/yr (15-28W avg) | ~$30/yr | $500+/yr (servers) |
| **Monitoring** | $0 (journalctl/top) | $0 (docker logs/stats) | Included |
| **Year-1 Total** | **~$830** | **~$830** | **$200K-600K+** |

### 3-Year TCO

| Year | Open5GS | free5GC | Radisys |
|---|---|---|---|
| Year 1 | ~$830 | ~$830 | $200K-600K |
| Year 2 | ~$30 (electricity) | ~$30 | $150K-500K |
| Year 3 | ~$30 | ~$30 | $150K-500K |
| **3-Year Total** | **~$900** | **~$900** | **$500K-1.6M** |

```
3-Year TCO (log scale)

  Open5GS on NUC 15    █  ~$900
  free5GC on NUC 15    █  ~$900
  Radisys              ████████████████████████████████████████  ~$800K

  For a 20-UE lab, both OSS options cost the same. The NUC is the only expense.
```

---

## 19. Risk Register

| Risk | Open5GS | free5GC | Mitigation |
|---|:---:|:---:|---|
| **NUC 15 hardware failure** | Equal | Equal | Keep configs in git. Redeploy on new NUC in 30 min |
| **gtp5g kernel breakage** | N/A | **Medium** | Pin kernel version; DKMS auto-rebuild |
| **NUC 15 kernel update breaks gtp5g** | N/A | **Medium** | `apt-mark hold linux-image-*` or DKMS |
| **Lead maintainer leaves** | HIGH | Medium | Fork capability; growing community |
| **Critical CVE** | Self-patch (hours) | Self-patch (hours) | OSS advantage |
| **License dispute** | Medium (AGPLv3) | Low (Apache 2.0) | Legal review if commercializing |
| **Outgrow NUC 15** | Low | Low | Same configs → multi-node cluster |
| **Team turnover** | HIGH (C skills) | Medium (Go skills) | Document setup; automate with scripts |

At our lab scale, most risks are **low impact**. The biggest real risk is gtp5g kernel compatibility (free5GC only).

---

## 20. Decision Framework

### Choose Open5GS if:

- You want `apt install open5gs` simplicity — nothing simpler exists
- **Hybrid LTE + 5G** testing needed
- No kernel module hassle — works on any stock kernel
- Team has C and telecom experience
- You prefer bare metal (no Docker overhead at all)
- Copyleft license (AGPLv3) is acceptable

### Choose free5GC if:

- You like **Docker-based workflow** (single `docker-compose up`)
- Built-in **WebUI** for subscriber management is appealing
- **Apache 2.0 license** matters for future commercialization
- You need **N3IWF, CHF, NEF, or TNGF**
- Team has Go and container skills
- You accept the gtp5g kernel module dependency
- You want the option to scale to K8s later by splitting the image into per-NF pods

### Skip Radisys:

- Not viable for NUC deployment
- Not sensible for 20-UE lab
- $200K+ for what you get free with OSS
- Only consider if zero in-house 5G expertise AND budget allows it AND you need carrier-grade

### Decision Flowchart

```
START: 20-UE Lab on NUC 15?
  │
  ├─ Need hybrid 4G LTE + 5G?
  │    └─ YES ──────────────────────► Open5GS (only option)
  │
  ├─ Need N3IWF / WiFi offload / CHF?
  │    └─ YES ──────────────────────► free5GC (only option)
  │
  ├─ Want zero Docker dependency?
  │    └─ YES ──────────────────────► Open5GS (apt install)
  │
  ├─ Want built-in WebUI?
  │    └─ YES ──────────────────────► free5GC
  │
  ├─ Need Apache 2.0 license?
  │    └─ YES ──────────────────────► free5GC
  │
  ├─ Want simplest possible install?
  │    └─ YES ──────────────────────► Open5GS (apt install)
  │
  ├─ Prefer Docker workflow?
  │    └─ YES ──────────────────────► free5GC (single image)
  │
  └─ No strong preference?
       └─ Try both — takes 30 min each. At this scale, it doesn't matter.
```

---

## 21. PoC Playbook — NUC 15

### Bill of Materials

| Item | Cost | Notes |
|---|---|---|
| Intel NUC 15 Pro (barebones) | ~$700 | Arrow Lake Core Ultra |
| 32 GB DDR5 (2x16 GB) | ~$90 | DDR5-5600 |
| 512 GB NVMe Gen4 | ~$50 | |
| USB-C to GbE adapter (optional) | ~$20 | Only if external RAN |
| USRP B210 (optional, for OTA) | ~$1,500 | Only if real over-the-air |
| **Total (software lab)** | **~$840** | |
| **Total (OTA lab)** | **~$2,340** | |

### Quick Start: Open5GS on NUC 15 (30 minutes)

```
Step 1:  Install Ubuntu 22.04/24.04 LTS on NUC 15
Step 2:  Add Open5GS PPA and install
         $ sudo add-apt-repository ppa:open5gs/latest
         $ sudo apt update && sudo apt install open5gs
Step 3:  MongoDB is auto-installed as dependency
Step 4:  Edit configs: /etc/open5gs/*.yaml (set PLMN, TAC, DNN, IPs)
Step 5:  Restart all services
         $ sudo systemctl restart open5gs-*
Step 6:  Add test subscriber via mongosh:
         $ mongosh mongodb://localhost/open5gs --eval '
           db.subscribers.insertOne({
             imsi: "001010000000001",
             security: { k: "465B5CE8B199B49FAA5F0A2EE238A6BC",
                         opc: "E8ED289DEBA952E4283B54E88E6183CA" },
             ...
           })'
Step 7:  Install & run UERANSIM
         $ git clone https://github.com/aligungr/UERANSIM && cd UERANSIM && make
         $ ./nr-gnb -c config/open5gs-gnb.yaml
         $ ./nr-ue -c config/open5gs-ue.yaml
Step 8:  Verify: ping -I uesimtun0 8.8.8.8
```

### Quick Start: free5GC on NUC 15 (30 minutes)

```
Step 1:  Install Ubuntu 22.04/24.04 LTS on NUC 15
Step 2:  Install Docker
         $ curl -fsSL https://get.docker.com | sh
Step 3:  Build gtp5g kernel module
         $ sudo apt install linux-headers-$(uname -r)
         $ git clone https://github.com/free5gc/gtp5g && cd gtp5g
         $ make && sudo make install
Step 4:  Run free5GC (all-in-one CP + UPF + MongoDB)
         $ docker compose up -d
         (using free5GC docker-compose with single CP image)
Step 5:  Open WebUI at http://localhost:5000
         Default login: admin / free5gc
Step 6:  Add subscriber via WebUI (IMSI, key, OPc)
Step 7:  Run UERANSIM
         $ docker run --rm -it ueransim-gnb
         $ docker run --rm -it ueransim-ue
Step 8:  Verify: ping through UE tunnel
```

### PoC Timeline

```
Day 1 (morning): NUC 15 Setup
├── Unbox NUC 15, install DDR5 + NVMe
├── Install Ubuntu 22.04/24.04
└── Install prerequisites (Docker if using free5GC)

Day 1 (afternoon): Core Deployment
├── Deploy Open5GS (apt) OR free5GC (docker)
├── Configure PLMN, DNN, IP pools
├── Add test subscriber(s)
└── Verify all NFs are running

Day 2: Testing at Target Scale
├── 1 UE: basic registration + data plane
├── 20 UEs: simultaneous attach, PDU sessions
├── 200 UE identifications: bulk registration burst
├── 400 UE rejects: bad IMSI, wrong key, PLMN mismatch
└── Data plane: iperf through 20 tunnels

Day 3: Stability + Documentation
├── 24h soak test (20 UEs attached)
├── Repeat reject/ident cycles
├── Document findings
└── Present recommendation
```

### PoC Evaluation Scorecard

| Criterion | Weight | Open5GS | free5GC | Notes |
|---|:---:|:---:|:---:|---|
| Time to first UE attach | 15% | ___ | ___ | |
| Setup complexity on NUC 15 | 15% | ___ | ___ | |
| Stability over 24h soak | 20% | ___ | ___ | 20 UEs attached |
| 200-ident burst handling | 10% | ___ | ___ | |
| 400-reject burst handling | 10% | ___ | ___ | |
| Subscriber management UX | 10% | ___ | ___ | CLI vs WebUI |
| Team comfort level | 10% | ___ | ___ | |
| NF coverage for use case | 10% | ___ | ___ | |
| **Weighted Total** | **100%** | ___ | ___ | |

---

## 22. Governance Checklist

Before making a final decision, verify:

- [ ] **NUC 15 Pro** ordered — 32 GB DDR5, 512 GB NVMe minimum
- [ ] **Ubuntu 22.04/24.04** installed and updated
- [ ] **Kernel version** checked — compatible with gtp5g (if choosing free5GC)
- [ ] **Docker installed** (if choosing free5GC)
- [ ] **Target scale confirmed** — 20 attached, 200 ident, 400 reject
- [ ] **NF coverage** — all required NFs present in chosen platform
- [ ] **License compatibility** — reviewed against future plans
- [ ] **Team skills** — assessed C vs Go alignment
- [ ] **Subscriber provisioning** — decided on CLI (Open5GS) vs WebUI (free5GC)
- [ ] **Backup strategy** — config files in git, periodic mongodump
- [ ] **Scale-out plan** — documented path from NUC to cluster when needed

---

## 23. Appendix: Glossary

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
| **SBOM** | Software Bill of Materials | Inventory of software components |
| **TCO** | Total Cost of Ownership | Full cost including hidden/indirect costs |
| **PoC** | Proof of Concept | Validation deployment before production |
| **k3s** | Lightweight Kubernetes | CNCF-certified K8s for edge/IoT |
| **gtp5g** | GTP-U Kernel Module | Linux kernel module for GTP user plane |
| **DKMS** | Dynamic Kernel Module Support | Framework for auto-rebuilding kernel modules |
| **SDR** | Software Defined Radio | Radio hardware for OTA testing (e.g., USRP) |
| **USRP** | Universal Software Radio Peripheral | Ettus Research SDR platform |
| **DDR5** | Double Data Rate 5 | Latest generation system memory |
| **NVMe** | Non-Volatile Memory Express | High-speed SSD interface |

---

*This document is optimized for non-commercial lab/PoC deployment of a private 5G core on a single Intel NUC 15 Pro, targeting ~20 attached UEs with 200 identification and 400 reject test scenarios.*

*Generated: February 2026*
