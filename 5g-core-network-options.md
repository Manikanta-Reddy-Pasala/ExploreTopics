# 5G Core Network Comparison: Open5GS vs free5GC vs Radisys

> **Scope:** Lab / PoC / Private 5G — Non-Commercial Grade
> **Scale:** ~20 attached UEs, ~200 UE identifications, ~400 UE rejects
> **Last Updated:** February 2026

---

## 1. Executive Summary

| | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **One-liner** | Lean C core — full 5GC + EPC, bare-metal simplicity | Go core — all CP in a single Docker image, Apache 2.0 | Binary black-box — only IPF source, 15 mandatory frameworks |
| **Best for** | Hybrid LTE+5G, `apt install` simplicity | Docker workflow, permissive license, broader NF coverage | Zero in-house expertise, carrier-grade SLA requirement |
| **Source access** | Full (AGPLv3) | Full (Apache 2.0) | **IPF only** — all NFs are binaries |
| **Language** | C | Go | Undisclosed (binaries) |
| **Lab deployment** | `apt install open5gs` | `docker compose up` (single CP image + UPF) | 20+ containers (9 NFs + 15 frameworks) |
| **Year-1 cost** | ~$830 | ~$830 | $200K+ |

### Quick Decision

| If you need... | Go with |
|---|---|
| Simplest possible setup | **Open5GS** (`apt install`) |
| Hybrid LTE + 5G | **Open5GS** (only option) |
| Single Docker image for all CP | **free5GC** |
| Apache 2.0 license | **free5GC** |
| N3IWF, CHF, NEF, TNGF | **free5GC** |
| No kernel module dependency | **Open5GS** |
| Vendor SLA (with patience and budget) | **Radisys** |

---

## 2. Architecture

### 2.1 Open5GS

```
┌─ Open5GS (systemd services, bare metal) ─────────────────────┐
│                                                                │
│  5G Core (SBA)                 EPC (4G)                        │
│  AMF │ SMF │ UPF │ NRF        MME │ HSS │ SGW │ PGW           │
│  UDM │ UDR │ AUSF │ PCF       PCRF                             │
│  NSSF │ SCP                                                    │
│                                                                │
│  UPF: userspace (no kernel module needed)                      │
│  Config: /etc/open5gs/*.yaml                                   │
│  Install: apt install open5gs                                  │
└────────────────────────────────────────────────────────────────┘
```

- Each NF is a lightweight C binary managed by systemd
- **Unique advantage:** Full EPC (4G) + 5GC in one codebase
- UPF uses userspace libgtpnl + TUN — works on any stock kernel
- MongoDB for subscriber data

### 2.2 free5GC

```
┌─ free5GC (Single Docker Container — ALL CP NFs) ─────────────┐
│                                                                │
│  One container: AMF │ SMF │ NRF │ UDM │ UDR │ AUSF │ PCF     │
│                 NSSF │ N3IWF │ CHF │ NEF │ WebUI              │
│                                                                │
│  SBI via localhost (all NFs share one process space)           │
└────────────────────────────────────────────────────────────────┘
┌─ UPF (separate container, requires gtp5g kernel module) ──────┐
└────────────────────────────────────────────────────────────────┘
┌─ MongoDB ─┐
└────────────┘

Total: 3 containers (CP + UPF + MongoDB)
```

- All control plane NFs in a **single Docker image** — not 10-14 separate pods
- Built-in WebUI on port 5000 for subscriber management
- gtp5g kernel module required for UPF (must compile against host kernel)
- Can scale to per-NF K8s pods later without changing core

### 2.3 Radisys — Vendor Reality

```
┌─ Source Code Access (1 of ~20 components) ────────────────────┐
│  IPF (Interoperability Framework)                              │
│  └── The ONLY codebase you can read, modify, or debug          │
└────────────────────────────────────────────────────────────────┘

┌─ Binary-Only (everything else) ───────────────────────────────┐
│                                                                │
│  9 NFs: AMF │ SMF │ UPF │ NRF │ UDM │ UDR │ AUSF │ PCF │ NSSF│
│                                                                │
│  15 Mandatory Framework Components:                            │
│  Service mesh, logging, config mgmt, health checks,           │
│  orchestration layers — ALL required, NONE removable.          │
│  Every NF depends on all 15.                                   │
│                                                                │
│  Docker images: 2-5+ GB EACH (asked to reduce → "takes time") │
│  Total image pull: 30-50 GB                                    │
└────────────────────────────────────────────────────────────────┘
```

**What you can and cannot do:**

| Aspect | Reality |
|---|---|
| Source code | **IPF only** — one component out of ~20 |
| NF logic | Pre-compiled binaries — cannot read, audit, or modify |
| Frameworks | 15 mandatory — cannot remove any, even for a basic lab |
| Image sizes | 2-5+ GB per NF; optimization requested → "takes time" |
| Bug in an NF | Open ticket → wait for vendor (days to weeks) |
| Custom behavior | Not possible — no hooks, no extension points in NFs |
| Debugging | Logs only — cannot step through binary code |

---

## 3. Network Function Coverage

| Network Function | Role | Open5GS | free5GC | Radisys |
|---|---|:---:|:---:|:---:|
| AMF | Access & Mobility | Yes | Yes | Yes (binary) |
| SMF | Session Management | Yes | Yes | Yes (binary) |
| UPF | User Plane | Yes (userspace) | Yes (gtp5g kernel) | Yes (binary) |
| NRF | NF Discovery | Yes | Yes | Yes (binary) |
| UDM / UDR / AUSF | Auth & Data | Yes | Yes | Yes (binary) |
| PCF | Policy Control | Yes | Yes | Yes (binary) |
| NSSF | Slice Selection | Yes | Yes | Binary — claimed |
| SCP | Service Proxy | Yes | Partial | Unknown |
| NEF | Network Exposure | Partial | **Yes** | Binary — claimed |
| N3IWF | WiFi Offload | No | **Yes** | Binary — claimed |
| TNGF | Trusted Non-3GPP | No | **Yes** | Unknown |
| CHF | Charging | No | **Yes** | Binary — claimed |
| EPC (4G) | MME, SGW, PGW, HSS | **Yes (full)** | No | Separate product |
| WebUI | Subscriber Mgmt | Community tools | **Built-in** | Vendor portal |

**Bottom line:** Both OSS platforms cover the essential NFs. free5GC wins on breadth (N3IWF, CHF, NEF, TNGF). Open5GS wins on hybrid 4G+5G.

---

## 4. Resource Footprint

At 20 attached UEs, both OSS options are trivially lightweight. One section covers it.

| Component | Open5GS | free5GC | Radisys |
|---|---|---|---|
| All CP NFs | ~300-500 MB | ~500 MB - 1 GB | ~4-8 GB |
| 15 Frameworks | N/A | N/A | ~4-8 GB (mandatory) |
| UPF | ~50-100 MB | ~100-200 MB | ~1-2 GB |
| MongoDB | ~200 MB | ~200 MB | ~200 MB |
| **Total RAM** | **~600 MB - 1 GB** | **~1-1.5 GB** | **~10-18 GB** |
| CPU (steady) | ~0.2 core | ~0.2 core | ~2-4 cores |
| Docker images on disk | ~200 MB | ~500 MB | **30-50 GB** |

At this scale, Open5GS and free5GC are functionally identical in resource consumption. Radisys framework overhead dominates — 8-16 GB RAM consumed before a single UE attaches.

---

## 5. Developer Experience

| Aspect | Open5GS | free5GC | Radisys |
|---|---|---|---|
| Install | `apt install open5gs` | `docker compose up` | Vendor PS + days of setup |
| Add subscriber | `mongosh` CLI | WebUI (port 5000) | Vendor portal |
| View logs | `journalctl -u open5gs-amfd` | `docker logs free5gc-cp` | 20+ container logs |
| Fix a bug | Read source → fix → rebuild | Read source → fix → rebuild | Logs → ticket → wait |
| Upgrade | `apt upgrade` | `docker pull` new tag | Pull 30-50 GB of images |
| Debug a crash | GDB + core dump + full source | Delve + goroutine traces | Logs only — NFs are opaque |
| Build time | ~2 min (Meson + Ninja) | ~1 min (`go build`) | IPF only; NFs are pre-built |
| Hiring pool | Harder (C + telecom) | Easier (large Go pool) | Radisys-specific tribal knowledge |
| Onboarding | 4-8 weeks | 2-4 weeks | Weeks + limited by binary opacity |

---

## 6. Security & Supply Chain

| Control | Open5GS | free5GC | Radisys |
|---|---|---|---|
| Build from source | Yes | Yes | IPF only |
| SBOM generation | Full | Full | **None** — cannot inspect binaries |
| CVE scanning | Full (Trivy, Grype) | Full | Limited — cannot scan binary internals |
| Patch turnaround | Self-fix (hours) | Self-fix (hours) | Vendor queue (weeks/months) |
| Full source audit | Yes | Yes | **No** — only IPF auditable |

```
Supply Chain Control Spectrum

  FULL CONTROL ◄──────────────────────────────────► ZERO CONTROL
  ┌──────────┐  ┌──────────┐         ┌──────────────────────────┐
  │ Open5GS  │  │ free5GC  │         │ Radisys                  │
  │ Full src │  │ Full src │         │ IPF: source (1 of ~20)   │
  │ Full SBOM│  │ Full SBOM│         │ Everything else: binary  │
  └──────────┘  └──────────┘         │ No SBOM, no audit        │
                                     └──────────────────────────┘
```

---

## 7. Licensing

| Scenario | Open5GS (AGPLv3) | free5GC (Apache 2.0) | Radisys (Proprietary) |
|---|---|---|---|
| Internal lab | Safe | Safe | Per contract |
| Modifications for internal use | Safe | Safe | Per contract |
| Offering as network service | **Must share modified source** | No obligation | N/A |
| Embedding in commercial product | **Copyleft — must open-source** | Permissive — attribution only | Per contract |
| Forking for proprietary use | Not allowed | **Allowed** | N/A |

If there's any chance of future commercialization, **free5GC's Apache 2.0** is the safer choice.

---

## 8. 3GPP Compliance

| Aspect | Open5GS | free5GC | Radisys |
|---|---|---|---|
| Target Release | Rel-17 (partial Rel-18) | Rel-15/16 (some Rel-17) | Vendor-claimed Rel-16+ |
| SBI (HTTP/2) | Yes | Yes | Vendor impl |
| NAS / NGAP / PFCP | Full | Full | Vendor |
| UPF data path | Userspace (portable) | Kernel module (faster) | Vendor |
| Interop tested | UERANSIM, srsRAN | UERANSIM, srsRAN | Vendor lab |

Both OSS options are 3GPP-compliant for registration, auth, PDU sessions, and data plane at lab scale.

---

## 9. Open Source Health

| Metric | Open5GS | free5GC |
|---|---|---|
| GitHub Stars | ~2,800+ | ~2,400+ |
| Contributors | ~120+ | ~50+ |
| Active Since | 2017 (as NextEPC) | 2019 |
| Primary Backer | Community (Sukchan Lee) | Academic (NCTU/NYCU) |
| Key Person Risk | **High** (Sukchan Lee) | Medium (NYCU team) |
| Mitigation | Forkable (AGPLv3) | Forkable (Apache 2.0) |

---

## 10. Day-2 Operations

| Operation | Open5GS | free5GC | Radisys |
|---|---|---|---|
| Restart one NF | `systemctl restart open5gs-amfd` | Restart container | Orchestrated restart (framework deps) |
| Config change | Edit YAML → restart service | Edit mounted YAML → restart container | Vendor config portal |
| Rollback | `apt install open5gs=<ver>` | `docker run` with old tag | Revert all 20+ images |
| Backup | `mongodump` + git configs | Same | `mongodump` + vendor config export |
| Monitoring (lab) | `journalctl` + `top` | `docker stats` + `docker logs` | Need monitoring for 20+ containers |
| Image pull | Seconds (~200 MB) | Seconds (~500 MB) | 30-60 min (30-50 GB) |

No Prometheus/Grafana needed at 20 UEs. `journalctl` and `docker logs` are sufficient.

---

## 11. Resilience (Lab)

| Aspect | Open5GS | free5GC |
|---|---|---|
| Auto-restart | systemd `Restart=always` | Docker `restart: always` |
| DB backup | Periodic `mongodump` | Same |
| Full failure recovery | Configs in git → redeploy in 30 min | `docker-compose.yaml` in git → redeploy in 30 min |
| Kernel pinning | Not needed | **Recommended** (avoid gtp5g breakage) |

---

## 12. Testing Ecosystem

| Tool | Type | Open5GS | free5GC |
|---|---|:---:|:---:|
| UERANSIM | UE + gNB simulator | Tested | Tested |
| srsRAN | Open-source RAN | Tested | Tested |
| PacketRusher | Go-based load tester | Compatible | Compatible |
| Baicells / Airspan | Commercial small cells | Community tested | Community tested |

### Test Plan

| Test | What | Tool |
|---|---|---|
| Basic attach | 1 UE: register + PDU + ping | UERANSIM |
| Multi-UE | 20 UEs simultaneous | UERANSIM |
| Identification burst | 200 registrations | PacketRusher |
| Reject scenarios | 400 rejects (bad IMSI, wrong key, PLMN mismatch) | UERANSIM |
| Data plane | iperf through 20 tunnels | iperf3 |
| Soak test | 24-48h with 20 UEs attached | UERANSIM |

---

## 13. Total Cost of Ownership

| Component | Open5GS | free5GC | Radisys |
|---|---|---|---|
| Hardware | ~$800 | ~$800 | Server infrastructure |
| Software license | $0 | $0 | $100K-500K+/yr |
| Setup | 30 min (self) | 30 min (self) | Weeks + vendor PS |
| **Year-1** | **~$830** | **~$830** | **$200K+** |
| **3-Year** | **~$900** | **~$900** | **$500K-1.6M** |

Hidden Radisys costs: team time debugging opaque binaries, weeks waiting for image reductions, 30-50 GB disk for images, vendor lock-in.

---

## 14. Risk Register

| Risk | Open5GS | free5GC | Radisys |
|---|---|---|---|
| gtp5g kernel breakage | N/A | Medium (pin kernel) | N/A |
| Lead maintainer leaves | High (fork possible) | Medium (fork possible) | N/A |
| Critical CVE | Self-patch in hours | Self-patch in hours | Wait for vendor |
| CVE in binary component | N/A | N/A | **High** — cannot scan or patch |
| Vendor unresponsive | N/A | N/A | **High** — stuck with current binaries |
| Framework component failure | N/A | N/A | **High** — 1 of 15 fails → 5GC down |
| Cannot reproduce a bug | Low | Low | **High** — no source, logs only |

---

## 15. Migration Paths

| Path | Effort | Notes |
|---|---|---|
| Open5GS → free5GC | 1-2 days | MongoDB subscriber data is portable. Install gtp5g. |
| free5GC → Open5GS | 1 day | Remove gtp5g. Lose WebUI / N3IWF. |
| Single Docker → K8s (free5GC) | 1 day | Split into per-NF Helm chart. Same configs. |
| Either OSS → Radisys | 4-8 weeks | Complete re-provisioning. Vendor PS required. |
| Radisys → Either OSS | 1-2 weeks | Subscriber data portable. Must rebuild configs. |

Start with whichever OSS feels right. At 20 UEs, switching takes a day or two.

---

## 16. Decision Framework

### Choose Open5GS if:
- You want `apt install` — nothing simpler
- Hybrid LTE + 5G testing needed
- No kernel module hassle — stock kernel works
- Team has C / telecom experience
- Copyleft (AGPLv3) is acceptable

### Choose free5GC if:
- Docker workflow preferred (`docker compose up`)
- Built-in WebUI for subscriber management
- Apache 2.0 license matters
- Need N3IWF, CHF, NEF, or TNGF
- Team has Go / container skills
- Want option to scale to K8s later

### Why Radisys Doesn't Fit (From Real Experience):
- **Only IPF source code** — every other component is binary
- **15 mandatory frameworks** — cannot remove any; every NF depends on all 15
- **2-5 GB per NF image** — 30-50 GB total vs ~500 MB for OSS
- **Framework overhead** — 8-16 GB RAM before a single UE attaches
- **Slow vendor velocity** — image reduction requested → "takes time"
- **Blind debugging** — NF crash = logs + vendor ticket
- **$200K+ for what OSS provides free** with full source and lighter images
- Only consider if: zero in-house 5G expertise AND unlimited budget AND carrier-grade SLA is contractually required

```
Decision Flowchart:

Need hybrid 4G + 5G?  ──── YES ──► Open5GS
Need N3IWF / CHF?     ──── YES ──► free5GC
Want zero Docker?     ──── YES ──► Open5GS
Want built-in WebUI?  ──── YES ──► free5GC
Need Apache 2.0?      ──── YES ──► free5GC
No strong preference? ──── Try both (30 min each)
```

---

## 17. Quick Start

### Open5GS (30 minutes)

```
1. Install Ubuntu 22.04/24.04
2. sudo add-apt-repository ppa:open5gs/latest
   sudo apt update && sudo apt install open5gs
3. Edit /etc/open5gs/*.yaml (PLMN, TAC, DNN, IPs)
4. sudo systemctl restart open5gs-*
5. Add subscriber via mongosh
6. Install UERANSIM → run gNB + UE → ping through tunnel
```

### free5GC (30 minutes)

```
1. Install Ubuntu 22.04/24.04 + Docker
2. Build gtp5g kernel module:
   git clone https://github.com/free5gc/gtp5g && cd gtp5g
   make && sudo make install
3. docker compose up -d (single CP image + UPF + MongoDB)
4. Open WebUI at localhost:5000 (admin / free5gc)
5. Add subscriber via WebUI
6. Run UERANSIM → ping through tunnel
```

### PoC Timeline

```
Day 1: Setup + deploy chosen platform + verify 1 UE
Day 2: Test at scale (20 attach, 200 ident, 400 reject, iperf)
Day 3: 24h soak test + document findings
```

---

## Glossary

| Term | Meaning |
|---|---|
| AMF | Access and Mobility Management Function |
| SMF | Session Management Function |
| UPF | User Plane Function |
| NRF | NF Repository Function |
| UDM | Unified Data Management |
| UDR | Unified Data Repository |
| AUSF | Authentication Server Function |
| PCF | Policy Control Function |
| NSSF | Network Slice Selection Function |
| NEF | Network Exposure Function |
| N3IWF | Non-3GPP Interworking Function (WiFi offload) |
| CHF | Charging Function |
| SCP | Service Communication Proxy |
| SBI | Service-Based Interface (HTTP/2) |
| EPC | Evolved Packet Core (4G) |
| GTP-U | GPRS Tunnelling Protocol — User plane |
| NGAP | Next Generation Application Protocol (N2) |
| PFCP | Packet Forwarding Control Protocol (N4) |
| PDU | Protocol Data Unit (data session) |
| UERANSIM | Open-source 5G UE + gNB simulator |
| gtp5g | Linux kernel module for GTP user plane (free5GC) |
| SBOM | Software Bill of Materials |

---

*Generated: February 2026*
