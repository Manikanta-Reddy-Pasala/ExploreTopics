# 5G Core Network Comparison: Open5GS vs free5GC vs Radisys

> **Document Type:** Technical Evaluation & Decision Guide
> **Scope:** Private 5G / Lab / Enterprise deployment (non Tier-1 carrier grade)
> **Audience:** Platform Engineering, CTO, Network Architects
> **Last Updated:** February 2026
> **Status:** ACTIVE - Review quarterly

---

## Table of Contents

1. [Executive Summary & Verdict](#1-executive-summary--verdict)
2. [At a Glance](#2-at-a-glance)
3. [Architecture Deep Dive](#3-architecture-deep-dive)
4. [Network Function Coverage Matrix](#4-network-function-coverage-matrix)
5. [Language, Code Quality & Developer Experience](#5-language-code-quality--developer-experience)
6. [Container Footprint & Resource Planning](#6-container-footprint--resource-planning)
7. [3GPP Standards Compliance](#7-3gpp-standards-compliance)
8. [Open Source Health & Governance](#8-open-source-health--governance)
9. [Security & Supply Chain](#9-security--supply-chain)
10. [Licensing Deep Dive](#10-licensing-deep-dive)
11. [Day-2 Operations](#11-day-2-operations)
12. [Scaling & High Availability](#12-scaling--high-availability)
13. [Testing & Simulation Ecosystem](#13-testing--simulation-ecosystem)
14. [RAN Interoperability](#14-ran-interoperability)
15. [Migration Paths](#15-migration-paths)
16. [Total Cost of Ownership](#16-total-cost-of-ownership)
17. [Risk Register](#17-risk-register)
18. [Decision Framework](#18-decision-framework)
19. [PoC Playbook](#19-poc-playbook)
20. [Governance Checklist](#20-governance-checklist)
21. [Appendix: Glossary](#21-appendix-glossary)

---

## 1. Executive Summary & Verdict

### The Short Version

| | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **One-liner** | Lean, battle-tested C core with hybrid 4G/5G | Cloud-native Go core with broadest NF coverage | Turnkey vendor-packaged black-box |
| **Best for** | Resource-constrained / hybrid LTE+5G | K8s-native / commercial embedding | Zero in-house telecom expertise |
| **License** | AGPLv3 (copyleft) | Apache 2.0 (permissive) | Proprietary |
| **Language** | C | Go | Undisclosed |
| **Maturity** | High (since 2017) | Medium-High (since 2019) | Vendor-dependent |
| **Year-1 TCO** | $50K-100K | $60K-120K | $200K-600K+ |

### Quick Decision Matrix

| If your priority is... | Recommendation |
|---|---|
| Kubernetes-native, microservice-first deployment | **free5GC** |
| VM-based lab, hybrid LTE+5G, minimal container sprawl | **Open5GS** |
| Minimal internal engineering, vendor SLA required | **Radisys** |
| Maximum control over NFs and supply chain | **Open5GS or free5GC** |
| Fastest time-to-deploy with zero telecom expertise | **Radisys** |
| Apache 2.0 license for commercial product embedding | **free5GC** |
| Lowest resource footprint (edge / IoT) | **Open5GS** |
| Broadest NF coverage (N3IWF, CHF, NEF, TNGF) | **free5GC** |

---

## 2. At a Glance

### Scorecard (1-5, higher is better)

| Criteria | Open5GS | free5GC | Radisys | Notes |
|---|:---:|:---:|:---:|---|
| **Deployment Simplicity** | 4 | 3 | 4 | Open5GS runs on a single VM; Radisys is vendor-guided |
| **Cloud-Native Readiness** | 2 | 5 | 3 | free5GC designed for K8s from the ground up |
| **Resource Efficiency** | 5 | 3 | 2 | Open5GS has the smallest footprint |
| **NF Coverage Breadth** | 3 | 5 | 3 | free5GC implements the most NFs |
| **4G/5G Hybrid Support** | 5 | 1 | 2 | Open5GS uniquely supports EPC + 5GC in one codebase |
| **Code Auditability** | 5 | 5 | 0 | OSS = full audit; Radisys = black box |
| **Customizability** | 5 | 5 | 1 | Source access vs vendor request queue |
| **Vendor Support / SLA** | 1 | 1 | 5 | Only Radisys offers contractual SLA |
| **Hiring Pool** | 2 | 4 | N/A | Go devs >> C+telecom specialists |
| **License Flexibility** | 2 | 5 | 1 | Apache 2.0 is most business-friendly |
| **Supply Chain Control** | 5 | 5 | 0 | Full SBOM, image signing, CVE scanning for OSS |
| **Community & Ecosystem** | 4 | 4 | 1 | Both OSS have active, growing communities |
| **Operational Maturity** | 3 | 3 | 4 | Vendor handles Day-2 for Radisys |
| **Total (out of 65)** | **46** | **49** | **26** | |

> **Interpretation:** For teams with engineering capability, OSS options score nearly 2x the vendor option. The trade-off is engineering effort vs. vendor dependence.

---

## 3. Architecture Deep Dive

### 3.1 Open5GS

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

**Architecture Philosophy:** Monolithic-leaning, spec-faithful. Each NF is a lightweight C binary (single process). Uniquely supports **both EPC (4G) and 5GC** in a single codebase — the only open-source option that does this.

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

**Strengths:**
- Lowest resource footprint of any open-source 5GC
- Hybrid 4G/5G — no separate EPC stack needed for migration scenarios
- Mature codebase (~5,000+ commits, active since 2017 as NextEPC)
- Direct 3GPP spec mapping in C — straightforward spec compliance tracing
- Can run entire core on a **single node** for lab/dev
- Consistent release cadence from lead maintainer (Sukchan Lee)

**Weaknesses:**
- HA and horizontal scaling require custom engineering (no built-in clustering)
- C codebase raises the contributor bar (memory management, debugging complexity)
- Less cloud-native patterning (no native service mesh integration)
- Bus factor risk on lead maintainer
- No built-in WebUI for subscriber management

---

### 3.2 free5GC

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

**Architecture Philosophy:** True Service-Based Architecture per 3GPP spec. Each NF is a separate Go binary designed for independent containerized deployment. Kubernetes-first mindset.

| Property | Detail |
|---|---|
| **NF Design** | Separate Go binary per NF, each in its own container |
| **Inter-NF Communication** | SBI (HTTP/2) — strict 3GPP SBA |
| **Data Store** | MongoDB (shared or per-NF) |
| **Configuration** | YAML files per NF |
| **Build System** | Standard Go modules (go build) |
| **External Dependencies** | gin, mongo-driver, and others — larger dep tree |
| **Deployment Model** | Kubernetes-first (Docker Compose also available) |
| **Typical Container Count** | 10-14 for full deployment |
| **Special Requirement** | **gtp5g kernel module** on UPF host (kernel 5.x+) |

**Strengths:**
- True cloud-native — each NF independently scalable via K8s HPA
- Go language — memory-safe, large hiring pool, excellent tooling (delve, pprof)
- Apache 2.0 license — most permissive, no copyleft concerns for commercial use
- Broadest NF coverage: N3IWF, TNGF, CHF, NEF all implemented
- Built-in WebUI for subscriber management
- Academic backing (NCTU/NYCU) with industry collaboration
- Clean module separation — can replace individual NFs

**Weaknesses:**
- More containers = more orchestration overhead and resource consumption
- **gtp5g kernel module** is a hard dependency for UPF (limits deployment targets)
- Multi-repo structure — tracking cross-repo changes is harder
- Higher baseline resource consumption than Open5GS
- Some 3GPP edge cases may lag behind Open5GS
- No 4G/EPC support

---

### 3.3 Radisys 5G Core

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

**Architecture Philosophy:** Black-box, vendor-managed. Delivered as pre-built images with a management portal. Designed for customers who want a turnkey solution without internal telecom expertise.

| Property | Detail |
|---|---|
| **NF Design** | Vendor-packaged (opaque internals) |
| **Inter-NF Communication** | Vendor implementation |
| **Configuration** | Vendor portal / guided setup |
| **Build System** | N/A — pre-built images |
| **Deployment Model** | Vendor-guided (K8s or VM) |
| **Special Requirement** | Vendor professional services for initial setup |

**Strengths:**
- Turnkey — minimal internal telecom expertise required
- Vendor SLA and contractual support
- Potentially pre-certified for specific use cases
- Integrated RAN+Core story with Radisys Connect platform
- Vendor handles security patches and upgrade lifecycle

**Weaknesses:**
- **Black-box** — no source code visibility, cannot audit or independently patch
- **Vendor lock-in** — high switching cost, contractual exit terms
- Limited customization — feature requests go through vendor queue
- Significant cost: license + annual support + professional services
- Image opacity — cannot run SBOM analysis, slim images, or scan for CVEs
- Dependency on vendor's release cadence for bug fixes

---

## 4. Network Function Coverage Matrix

| Network Function | 3GPP Role | Open5GS | free5GC | Radisys |
|---|---|:---:|:---:|:---:|
| **AMF** | Access & Mobility Management | Yes | Yes | Yes |
| **SMF** | Session Management | Yes | Yes | Yes |
| **UPF** | User Plane Function | Yes | Yes (gtp5g) | Yes |
| **NRF** | NF Repository Function | Yes | Yes | Yes |
| **UDM** | Unified Data Management | Yes | Yes | Yes |
| **UDR** | Unified Data Repository | Yes | Yes | Yes |
| **AUSF** | Authentication Server | Yes | Yes | Yes |
| **PCF** | Policy Control Function | Yes | Yes | Yes |
| **NSSF** | Network Slice Selection | Yes | Yes | Claimed |
| **SCP** | Service Communication Proxy | Yes | Partial | Unknown |
| **NEF** | Network Exposure Function | Partial | Yes | Claimed |
| **N3IWF** | Non-3GPP Interworking | No | **Yes** | Claimed |
| **TNGF** | Trusted Non-3GPP Gateway | No | **Yes** | Unknown |
| **CHF** | Charging Function | No | **Yes** | Claimed |
| **EPC (4G)** | MME, SGW, PGW, HSS, PCRF | **Yes (full)** | No | Separate product |
| **WebUI** | Subscriber Management | Community tools | **Built-in** | Vendor portal |

**Key Takeaways:**
- **free5GC** has the broadest 5G NF coverage (N3IWF, TNGF, CHF, NEF)
- **Open5GS** uniquely supports **hybrid 4G EPC + 5GC** in a single codebase
- **Radisys** claims are unverifiable without source access or independent testing
- For non-3GPP access (WiFi offload via N3IWF) — free5GC is the only proven OSS option

---

## 5. Language, Code Quality & Developer Experience

### Side-by-Side Comparison

| Aspect | Open5GS (C) | free5GC (Go) | Radisys (Proprietary) |
|---|---|---|---|
| **Memory Safety** | Manual — risk of buffer overflows, use-after-free | GC-managed — no manual memory bugs | Unknown — cannot audit |
| **Performance** | Excellent — deterministic latency, zero GC pauses | Good — slight GC overhead, adequate for private 5G | Vendor-benchmarked (unverifiable) |
| **Concurrency Model** | Event-driven (epoll/kqueue) | Goroutines — lightweight, easy to reason about | Unknown |
| **Testability** | Unit tests exist; fewer modern C test frameworks | Go built-in `testing` package, easy unit/integration tests | Internal QA only |
| **Code Structure** | Well-organized, consistent style, spec-traceable | Clean per-NF separation, standard Go project layout | N/A |
| **Build System** | Meson + Ninja — fast, reliable | `go build` — simple, reproducible | N/A |
| **Dependency Count** | Minimal (talloc, mongoc, yaml, nghttp2) | More deps (gin, mongo-driver, etc.) — larger attack surface | Unknown |
| **Debugging** | GDB, Valgrind, AddressSanitizer — powerful but steep | Delve, pprof — modern and accessible | Vendor support tickets |
| **IDE Support** | Good (CLion, VS Code with C extensions) | Excellent (GoLand, VS Code with Go extension) | N/A |
| **Hiring Difficulty** | **High** — need C + telecom intersection | **Low-Medium** — large Go developer pool | N/A (vendor staff) |
| **Onboarding Time** | 4-8 weeks for effective contribution | 2-4 weeks for effective contribution | N/A |

### Developer Workflow Comparison

| Workflow | Open5GS | free5GC |
|---|---|---|
| **Clone to first build** | `meson build && ninja -C build` (~2 min) | `go build ./...` (~1 min) |
| **Run unit tests** | `ninja -C build test` | `go test ./...` |
| **Add a new NF feature** | Implement in C, follow existing NF pattern | Implement in Go, follow existing NF pattern |
| **Debug a crash** | GDB + core dump analysis | Delve + goroutine stack traces |
| **Profile performance** | `perf` + flamegraphs | `pprof` + built-in profiling |
| **Static analysis** | clang-tidy, cppcheck | `go vet`, `staticcheck`, `golangci-lint` |

---

## 6. Container Footprint & Resource Planning

### Baseline Resource Requirements

| Metric | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **NF Container Count** | 5-8 | 10-14 | Vendor-defined |
| **Typical Image Size (per NF)** | 30-80 MB (Alpine-based) | 80-200 MB (Go binaries) | 200 MB+ (opaque) |
| **Total Core Footprint** | ~300-600 MB | ~1-3 GB | Unknown (typically larger) |
| **Min RAM (lab/PoC)** | 2 GB | 4 GB | 8 GB+ |
| **Min CPU (lab/PoC)** | 2 cores | 4 cores | 4+ cores |
| **Min RAM (production, 1K UEs)** | 4-8 GB | 8-16 GB | Vendor-specified |
| **Min CPU (production, 1K UEs)** | 4 cores | 8 cores | Vendor-specified |
| **Storage (logs + DB)** | 10-20 GB | 20-50 GB | Vendor-specified |

### Image Optimization Potential

| Technique | Open5GS | free5GC | Radisys |
|---|---|---|---|
| Multi-stage Docker builds | Yes | Yes | N/A |
| Alpine / distroless base | Yes | Yes | N/A |
| Strip debug symbols | Yes | Yes (`-ldflags "-s -w"`) | N/A |
| Remove unused NFs | Yes | Yes | N/A |
| Pin exact dependency versions | Yes (`meson.build`) | Yes (`go.sum`) | N/A |
| Custom image registry | Yes | Yes | Vendor-specific |
| SBOM generation | Full (Syft, Trivy) | Full (Syft, Trivy) | Limited/None |
| CVE scanning | Full (Trivy, Grype) | Full (Trivy, Grype) | Vendor responsibility |

### Deployment Topology Examples

**Open5GS — Minimal Lab (1 VM)**
```
┌──────────────────────────────────┐
│  Single VM (4 CPU, 8 GB RAM)     │
│                                  │
│  AMF  SMF  UPF  NRF  UDM        │
│  UDR  AUSF PCF  NSSF            │
│  MongoDB                         │
│                                  │
│  + UERANSIM (test UE + gNB)     │
└──────────────────────────────────┘
```

**free5GC — K8s Cluster (3 nodes)**
```
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Node 1     │ │  Node 2     │ │  Node 3     │
│  (Control)  │ │  (Control)  │ │  (UPF Host) │
│             │ │             │ │             │
│  AMF, NRF   │ │  SMF, UDM   │ │  UPF        │
│  AUSF, PCF  │ │  UDR, NSSF  │ │  (gtp5g     │
│  NEF, CHF   │ │  N3IWF      │ │   kernel    │
│  WebUI      │ │  MongoDB    │ │   module)   │
└─────────────┘ └─────────────┘ └─────────────┘
```

---

## 7. 3GPP Standards Compliance

| Aspect | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Target 3GPP Release** | Release 17 (partial Rel-18) | Release 15/16 (some Rel-17) | Vendor-claimed Rel-16+ |
| **Spec Traceability** | Direct C → TS spec mapping | Go SBA → TS spec mapping | Opaque |
| **SBI Compliance** | HTTP/2 based SBI | HTTP/2 based SBI | Vendor implementation |
| **NAS Protocol** | Full 5G NAS | Full 5G NAS | Vendor |
| **NGAP (N2)** | Full | Full | Vendor |
| **PFCP (N4)** | Full | Full | Vendor |
| **GTP-U (N3/N9)** | Native implementation | gtp5g kernel module | Vendor |
| **Network Slicing** | Basic (NSSF) | NSSF + NEF | Vendor claim |
| **Policy & Charging** | PCF (basic) | PCF + CHF | Vendor claim |
| **Non-3GPP Access** | Not yet | N3IWF + TNGF | Vendor claim |
| **Interop Tested With** | UERANSIM, srsRAN, commercial gNBs | UERANSIM, srsRAN, commercial gNBs | Vendor lab |
| **Formal Certification** | No | No | Possible (vendor-provided) |

**Key Insights:**
- Open5GS tracks **higher 3GPP releases** (Rel-17+), meaning newer spec features
- free5GC has **broader NF coverage** within Rel-15/16
- If **formal certification documents** are a contractual requirement → Radisys
- If **spec traceability and auditability** matter → Open5GS or free5GC (you can read the source)

---

## 8. Open Source Health & Governance

### GitHub Metrics (as of early 2026)

| Metric | Open5GS | free5GC |
|---|---|---|
| **GitHub Stars** | ~2,800+ | ~2,400+ |
| **Forks** | ~1,000+ | ~900+ |
| **Total Commits** | ~5,000+ | ~500+ (main repo) |
| **Contributors** | ~120+ | ~50+ |
| **Active Since** | 2017 (as NextEPC) | 2019 |
| **Release Cadence** | Regular (monthly-ish) | Periodic |
| **Issue Response Time** | Days | Days to Weeks |
| **PR Review Process** | Maintainer-reviewed | Team-reviewed |
| **CI/CD** | GitHub Actions | GitHub Actions |
| **Documentation Site** | open5gs.org | free5gc.org |
| **Primary Backer** | Community (single maintainer led) | Academic (NCTU/NYCU) |

### Community Health Signals

| Signal | Open5GS | free5GC |
|---|---|---|
| Organic commit history | Yes — consistent, real development patterns | Yes — academic + community |
| Contributor diversity | Global contributors, consistent growth | NCTU/NYCU core team + external |
| Active issue triage | Yes | Yes |
| Release discipline | Tagged releases with changelogs | Tagged releases |
| Conference presence | FOSDEM, telecom meetups | Academic papers, 5G summits |
| Community channels | Discord, mailing list | Discord, forums |
| Signs of star-botting | No evidence | No evidence |

### Bus Factor Assessment

| | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Key Person Risk** | **HIGH** — Sukchan Lee is the primary maintainer | **MEDIUM** — academic team at NYCU | **LOW** — vendor organization |
| **Mitigation** | Forkable (AGPLv3); growing contributor base | Forkable (Apache 2.0); academic continuity | Contractual SLA; but vendor bankruptcy = high risk |
| **If project stalls** | Fork + community takeover feasible | Fork + community takeover feasible | No recourse — switch to OSS alternative |

---

## 9. Security & Supply Chain

| Control | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Build from source** | Yes | Yes | No |
| **Reproducible builds** | Yes (Meson) | Yes (Go modules) | No |
| **Sign your own images** | Yes (cosign, Notary) | Yes (cosign, Notary) | No |
| **Pin dependencies** | Yes (`meson.build`) | Yes (`go.sum`) | Vendor-dependent |
| **Remove unnecessary libs** | Yes (minimal base images) | Yes (scratch/distroless) | No |
| **SBOM generation** | Full (Syft, Trivy, etc.) | Full (Syft, Trivy, etc.) | Limited/None |
| **CVE scanning** | Full (Trivy, Grype, Snyk) | Full (Trivy, Grype, Snyk) | Vendor responsibility |
| **Patch turnaround** | **Self-controlled** (hours if critical) | **Self-controlled** (hours if critical) | Vendor SLA (days/weeks) |
| **Full source audit** | Yes | Yes | **No** |
| **Network policy control** | Full (you define firewall rules) | Full (you define firewall rules) | Vendor recommendations |
| **TLS/mTLS configuration** | Configurable per NF | Configurable per NF | Vendor-managed |
| **Secrets management** | Your choice (Vault, K8s secrets) | Your choice (Vault, K8s secrets) | Vendor-managed |

### Supply Chain Risk Summary

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

## 10. Licensing Deep Dive

| Scenario | Open5GS (AGPLv3) | free5GC (Apache 2.0) | Radisys (Proprietary) |
|---|---|---|---|
| **Internal use only** | Safe — no obligations triggered | Safe — no obligations triggered | Per contract terms |
| **Modifications for internal service** | Safe — internal use exception applies | Safe — no obligations | Per contract terms |
| **Offering as a network service** | **Must share modified source code** (AGPL network clause) | No obligation to share | N/A |
| **Embedding in commercial product** | **Copyleft applies — must open-source derivative work** | Permissive — no copyleft, just attribution | Per contract terms |
| **Forking for proprietary use** | **Not allowed** (AGPLv3 copyleft) | **Allowed** (Apache 2.0 is permissive) | N/A |
| **Mixing with proprietary code** | Must keep AGPLv3 boundary clear | No issue — permissive license | N/A |

> **Bottom Line:** If you plan to offer a 5G-based commercial service or embed the core in a product, **free5GC's Apache 2.0 license is significantly more business-friendly** than Open5GS's AGPLv3. Open5GS's AGPL triggers source-sharing obligations when the software is offered as a network service.

---

## 11. Day-2 Operations

### Deployment & Configuration

| Aspect | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Config Format** | YAML per NF | YAML per NF | Vendor portal/config |
| **Config Complexity** | Low-Medium | Medium-High (more NFs to configure) | Low (vendor handles) |
| **Helm Charts** | Community-maintained | Official + community | Vendor-provided |
| **Subscriber Provisioning** | MongoDB CLI / community scripts | **Built-in WebUI** | Vendor portal |
| **TLS/mTLS Setup** | Manual configuration | Manual configuration | Vendor-managed |
| **Multi-tenancy** | Custom engineering | Custom engineering | Vendor may support |

### Monitoring & Observability

| Aspect | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Prometheus Metrics** | Basic (community contributions) | Basic (community) | Vendor-specific |
| **Log Format** | Structured logs (configurable level) | Structured logs (configurable level) | Vendor format |
| **OpenTelemetry Tracing** | Community effort | Community effort | Vendor |
| **Health Checks** | Process-level | HTTP health endpoints | Vendor |
| **Grafana Dashboards** | Community-shared | Community-shared | Vendor portal |
| **Alerting** | DIY (Prometheus + Alertmanager) | DIY (Prometheus + Alertmanager) | Vendor portal |

### Upgrade & Rollback

| Aspect | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Upgrade Complexity** | Low-Moderate | Higher (more NFs to coordinate) | Vendor-managed |
| **Rolling Upgrades** | Manual orchestration | K8s-native rolling update per NF | Vendor procedure |
| **Rollback Strategy** | Container image pinning + DB backup | Container image pinning + DB backup | Vendor procedure |
| **DB Migration** | Manual (MongoDB schema changes) | Manual (MongoDB schema changes) | Vendor handles |
| **Breaking Changes** | Documented in release notes | Documented in release notes | Vendor communication |
| **Upgrade Frequency** | Monthly releases | Periodic releases | Vendor schedule |

---

## 12. Scaling & High Availability

| Aspect | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Horizontal Scaling** | Custom engineering required | K8s HPA possible per NF | Vendor-defined |
| **HA / Failover** | No built-in (external tools needed) | No built-in (external tools needed) | Vendor may provide |
| **Load Balancing** | External LB (HAProxy, NGINX) | External LB + SCP | Vendor |
| **Session Persistence** | Application-level engineering | Application-level engineering | Vendor |
| **Geographic Redundancy** | DIY | DIY | Vendor option |
| **Max Tested UEs (community)** | ~1,000-5,000 | ~1,000-5,000 | Vendor claims higher |
| **Scaling Bottleneck** | Single-process NFs; need custom sharding | UPF kernel module; control plane scales better | Unknown |

### Scaling Architecture Patterns

**Open5GS Scaling (Manual)**
```
                    ┌─────────┐
                    │   LB    │
                    └────┬────┘
              ┌──────────┼──────────┐
         ┌────▼───┐ ┌────▼───┐ ┌────▼───┐
         │ AMF-1  │ │ AMF-2  │ │ AMF-3  │
         └────────┘ └────────┘ └────────┘
         (Requires manual session routing)
```

**free5GC Scaling (K8s Native)**
```
         ┌─────────────────────────────┐
         │  K8s HPA + Service Mesh     │
         │                             │
         │  AMF: replicas 1→N (HPA)    │
         │  SMF: replicas 1→N (HPA)    │
         │  UPF: node-pinned (gtp5g)   │
         └─────────────────────────────┘
         (UPF scaling limited by kernel module)
```

---

## 13. Testing & Simulation Ecosystem

| Tool / Simulator | Description | Open5GS | free5GC | Radisys |
|---|---|:---:|:---:|:---:|
| **UERANSIM** | Open-source 5G UE + gNB simulator | Tested | Tested | Possible |
| **srsRAN** | Open-source 4G/5G RAN stack | Tested | Tested | Unknown |
| **PacketRusher** | Go-based 5G UE/gNB load tester | Compatible | Compatible | Unknown |
| **gNBSim** | ONF gNB simulator | Compatible | Compatible | Unknown |
| **my5G-RANTester** | 5G RAN protocol tester | Compatible | Compatible | Unknown |
| **Commercial UEs** | Quectel, Sierra, etc. | Community tested | Community tested | Vendor-validated |
| **Commercial gNBs** | Nokia, Ericsson, Baicells | Community reports | Community reports | Likely tested |

### Recommended Test Strategy

| Phase | Tool | Purpose |
|---|---|---|
| **Unit / NF-level** | UERANSIM | Basic registration, PDU session, deregistration |
| **Integration** | UERANSIM + multiple UEs | Multi-UE attach, concurrent sessions |
| **Load / Stress** | PacketRusher | 100-1000 simulated UEs, throughput measurement |
| **E2E with real RAN** | srsRAN + SDR | Real over-the-air testing |
| **Stability** | UERANSIM (48h soak) | Memory leaks, connection drops, recovery |

---

## 14. RAN Interoperability

| RAN Solution | Type | Open5GS | free5GC | Radisys |
|---|---|:---:|:---:|:---:|
| **UERANSIM** | Simulator | Tested | Tested | Tested |
| **srsRAN** | Open-source | Tested | Tested | Unknown |
| **Radisys Connect** | Commercial | Should work (3GPP) | Should work (3GPP) | **Native integration** |
| **Nokia** | Commercial | Community reports | Community reports | Likely tested |
| **Ericsson** | Commercial | Community reports | Community reports | Likely tested |
| **Baicells** | Commercial | Community tested | Community tested | Unknown |
| **Airspan** | Commercial | Community reports | Community reports | Unknown |
| **Compal/Sercomm** | Small cell | Community reports | Community reports | Unknown |

> **Note:** Any 3GPP-compliant gNB should interoperate with any 3GPP-compliant core via N2 (NGAP) and N3 (GTP-U). However, real-world interop quirks are common — always validate in a lab before production.

---

## 15. Migration Paths

| Migration Path | Difficulty | Effort | Key Considerations |
|---|:---:|---|---|
| **Open5GS → free5GC** | Medium | 2-4 weeks | Both use MongoDB — subscriber data is portable. NF configs differ. Lose 4G EPC capability. Gain N3IWF, CHF, WebUI. |
| **free5GC → Open5GS** | Medium | 2-4 weeks | Reverse of above. Lose NFs not in Open5GS (N3IWF, CHF, TNGF). Gain 4G EPC. |
| **Either OSS → Radisys** | High | 4-8 weeks | Full re-provisioning. No config portability. Vendor professional services required. |
| **Radisys → Either OSS** | **Very High** | 6-12 weeks | Need to extract subscriber data (vendor cooperation required). Full re-architecture. |

### Migration Risk Heat Map

```
                    Migration Difficulty
                    Low    Medium    High    Very High
  Open5GS → free5GC          ██
  free5GC → Open5GS          ██
  OSS → Radisys                        ████
  Radisys → OSS                                ██████
```

> **Strategic Advice:** Starting with OSS keeps your options open. Starting with Radisys creates a significant exit barrier.

---

## 16. Total Cost of Ownership

### Year-1 Cost Breakdown

| Cost Component | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Software License** | $0 | $0 | $100K-500K+/yr |
| **Infrastructure** (cloud/HW) | $2.4K-6K/yr | $3.6K-9.6K/yr | Vendor-specified |
| **Engineering — Setup** | 2-4 weeks senior eng ($10K-20K) | 2-4 weeks senior eng ($10K-20K) | 1-2 weeks integration ($5K-10K) |
| **Engineering — Customization** | 2-4 weeks ($10K-20K) | 2-4 weeks ($10K-20K) | Via vendor (feature request) |
| **Ongoing Maintenance** | 0.25-0.5 FTE ($25K-50K) | 0.25-0.5 FTE ($25K-50K) | Included in support contract |
| **Professional Services** | N/A | N/A | $20K-50K+ |
| **Annual Support Contract** | Community (free) | Community (free) | 15-20% of license ($15K-100K) |
| **Training** | Self-study + community | Self-study + community | Vendor training ($5K-10K) |
| **Year-1 Total** | **$50K-100K** | **$60K-120K** | **$200K-600K+** |

### 3-Year TCO Projection

| Year | Open5GS | free5GC | Radisys |
|---|---|---|---|
| Year 1 | $50K-100K | $60K-120K | $200K-600K |
| Year 2 | $30K-60K | $35K-70K | $150K-500K |
| Year 3 | $30K-60K | $35K-70K | $150K-500K |
| **3-Year Total** | **$110K-220K** | **$130K-260K** | **$500K-1.6M** |

> **Note:** OSS costs are dominated by engineering time. Radisys costs are dominated by license fees. As your team gains expertise, OSS Year-2+ costs drop significantly.

---

## 17. Risk Register

| Risk | Open5GS | free5GC | Radisys | Mitigation Strategy |
|---|:---:|:---:|:---:|---|
| **Lead maintainer leaves** | HIGH | Medium | Low | Fork immediately; invest in contributor diversity |
| **Critical security vuln** | Self-patch (hours) | Self-patch (hours) | Vendor SLA (days/weeks) | OSS advantage: immediate patching capability |
| **Feature gap** | Build it (C) | Build it (Go) | Vendor request queue | OSS: contribute upstream. Vendor: contract leverage. |
| **Vendor bankruptcy** | N/A | N/A | **HIGH** | Maintain OSS fallback plan; avoid deep vendor coupling |
| **License dispute** | Medium (AGPLv3 complex) | Low (Apache 2.0 clear) | Per contract | Legal review before deployment |
| **Performance ceiling** | Medium (C optimization) | Medium (GC tuning needed) | Unknown | Load test thoroughly before production |
| **Upgrade breaks production** | Medium | Higher (more NFs) | Low (vendor-tested) | Staging env + canary deployments + rollback plan |
| **Kernel module issues** | N/A | **HIGH** (gtp5g) | N/A | Pin kernel version; test gtp5g on target OS first |
| **Team turnover** | HIGH (niche C skills) | Medium (Go is common) | Low (vendor handles) | Document everything; cross-train team members |

---

## 18. Decision Framework

### Choose Open5GS if:

- VM-based or bare-metal deployment is preferred
- **Hybrid LTE + 5G** is a requirement (unique capability)
- Team has C and telecom protocol expertise
- Minimal container sprawl is desired
- Resource-constrained environment (edge, IoT gateway)
- Lowest possible footprint is critical
- You need the most mature codebase (since 2017)

### Choose free5GC if:

- **Kubernetes-native** deployment is the target architecture
- Team has Go and cloud-native expertise
- Long-term customization and per-NF scaling is expected
- **Apache 2.0 license** is needed for commercial embedding
- Broadest NF coverage is required (N3IWF, CHF, NEF, TNGF)
- DevOps maturity exists in the organization
- Built-in WebUI for subscriber management is valued

### Choose Radisys if:

- Internal telecom/5G expertise is **limited or absent**
- **Vendor SLA and contractual support** are hard requirements
- Budget allows $200K-600K+ Year-1 investment
- Integrated RAN+Core from a single vendor is preferred
- Time-to-deploy is the primary constraint (weeks, not months)
- Formal certification documentation is required
- Organization is willing to accept vendor lock-in trade-off

### Decision Flowchart

```
START
  │
  ├─ Do you have in-house telecom/5G engineering?
  │    │
  │    ├─ NO ──────────────────────────────► Radisys
  │    │                                     (or hire, then re-evaluate)
  │    │
  │    └─ YES
  │         │
  │         ├─ Do you need hybrid 4G + 5G?
  │         │    │
  │         │    └─ YES ───────────────────► Open5GS
  │         │
  │         ├─ Is Kubernetes your target platform?
  │         │    │
  │         │    └─ YES
  │         │         │
  │         │         ├─ Need Apache 2.0 license?
  │         │         │    │
  │         │         │    └─ YES ─────────► free5GC
  │         │         │
  │         │         └─ NO ───────────────► free5GC (still best for K8s)
  │         │
  │         ├─ Is minimal footprint critical?
  │         │    │
  │         │    └─ YES ───────────────────► Open5GS
  │         │
  │         └─ Default ────────────────────► Evaluate both OSS with PoC
```

---

## 19. PoC Playbook

### Minimum Viable PoC Setup

| Component | Open5GS PoC | free5GC PoC |
|---|---|---|
| **Infrastructure** | 1 VM (4 CPU, 8 GB RAM) | 3-node K8s (4 CPU, 8 GB RAM each) |
| **Operating System** | Ubuntu 22.04+ | Ubuntu 22.04+ (kernel 5.x+ for gtp5g) |
| **UE/gNB Simulator** | UERANSIM | UERANSIM |
| **Estimated Setup Time** | 1-2 weeks | 2-3 weeks |
| **Key Test Cases** | Registration, PDU session, handover, EPC attach | Registration, PDU session, slicing, N3IWF |
| **Success Criteria** | UE attach, data plane works, basic mobility | UE attach, data plane works, NF independent scaling |

### PoC Evaluation Scorecard

| Criterion | Weight | Open5GS (1-5) | free5GC (1-5) | Notes |
|---|:---:|:---:|:---:|---|
| Time to first UE attach | 15% | ___ | ___ | |
| Stability over 48h soak test | 20% | ___ | ___ | |
| Resource consumption (CPU/RAM) | 15% | ___ | ___ | |
| Configuration complexity | 10% | ___ | ___ | |
| Documentation quality | 10% | ___ | ___ | |
| Team comfort level | 15% | ___ | ___ | |
| NF coverage for your use case | 15% | ___ | ___ | |
| **Weighted Total** | **100%** | ___ | ___ | |

### PoC Timeline

```
Week 1: Setup + Basic Validation
├── Day 1-2: Infrastructure provisioning
├── Day 3-4: Core deployment + configuration
└── Day 5:   First UE attach test

Week 2: Functional Testing
├── Day 1-2: Multi-UE registration, PDU sessions
├── Day 3:   Data plane throughput testing
└── Day 4-5: Failure scenarios (NF restart, network partition)

Week 3 (free5GC only): K8s-Specific Testing
├── Day 1-2: HPA scaling test per NF
├── Day 3:   Rolling upgrade test
└── Day 4-5: 48h stability soak test

Week 3-4: Evaluation & Decision
├── Complete scorecard
├── Document findings
└── Present recommendation
```

---

## 20. Governance Checklist

Before making a final decision, verify the following:

- [ ] **Release cadence** — Evaluate release frequency over the last 12 months
- [ ] **CVE handling** — Verify security vulnerability response time
- [ ] **Bus factor** — Check number of active maintainers
- [ ] **Performance validation** — Load test under your expected subscriber count
- [ ] **Upgrade/rollback** — Test upgrade and rollback in staging
- [ ] **SBOM availability** — Verify for compliance requirements (especially vendor option)
- [ ] **NF coverage** — Confirm all required NFs are implemented (N3IWF, slicing, VoNR, charging)
- [ ] **License compatibility** — Legal review against your business model
- [ ] **Team skills** — Assess C vs Go vs vendor-managed alignment
- [ ] **3-year TCO** — Calculate including engineering time, not just license fees
- [ ] **RAN interop** — Test with your chosen gNB in lab
- [ ] **Monitoring integration** — Verify compatibility with your observability stack
- [ ] **Data migration path** — Document how to switch later if needed
- [ ] **Kernel compatibility** — Test gtp5g module on target OS (if choosing free5GC)
- [ ] **Regulatory requirements** — Check if formal certification is needed for your market

---

## 21. Appendix: Glossary

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
| **NGAP** | Next Generation Application Protocol | Protocol between gNB and AMF (N2 interface) |
| **PFCP** | Packet Forwarding Control Protocol | Protocol between SMF and UPF (N4 interface) |
| **PDU** | Protocol Data Unit | Data session between UE and network |
| **UERANSIM** | UE and RAN Simulator | Open-source 5G UE and gNB simulator |
| **HPA** | Horizontal Pod Autoscaler | K8s auto-scaling mechanism |
| **SBOM** | Software Bill of Materials | Inventory of software components |
| **TCO** | Total Cost of Ownership | Full cost including hidden/indirect costs |
| **PoC** | Proof of Concept | Validation deployment before production |

---

*This document should be reviewed and updated quarterly as the 5G open-source ecosystem evolves rapidly.*

*Generated: February 2026*
