# 5G Core Network Evaluation: Open5GS vs free5GC vs Radisys

> **Scope:** Private 5G / Lab / Enterprise deployment (non Tier-1 carrier grade)
> **Focus:** Architecture, code quality, container footprint, operational risk, supply chain, cost, and long-term maintainability
> **Last Updated:** February 2026

---

## 1. Executive Summary

| Criteria | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **License** | AGPLv3 | Apache 2.0 | Proprietary |
| **Language** | C | Go | Vendor internal (undisclosed) |
| **3GPP Release** | Rel-17 (partial Rel-18) | Rel-15 / Rel-16+ | Vendor-claimed Rel-16+ |
| **Container Footprint** | Low (5-8 NFs, ~50-150 MB per image) | Medium (10-14 NFs, ~100-300 MB per image) | High (vendor-packaged, opaque sizes) |
| **OSS Governance** | Community-driven (single maintainer led) | Linux Foundation adjacent / NCTU-backed | Vendor-controlled |
| **Modify & Rebuild** | Yes - full source | Yes - full source | No |
| **Supply Chain Control** | Full | Full | None |
| **Exit Risk** | Low (forkable, AGPLv3) | Low (forkable, Apache 2.0) | High (contractual lock-in) |
| **Architecture Complexity** | Moderate | Higher | Hidden |
| **Hiring Pool** | Niche (C + telecom) | Broader (Go developers) | N/A (vendor staff) |
| **Cost** | Free (engineering effort) | Free (engineering effort) | License + support fees |

### Quick Decision Matrix

| If you need... | Choose |
|---|---|
| Kubernetes-native, microservice-first deployment | **free5GC** |
| VM-based lab, hybrid LTE+5G, minimal container sprawl | **Open5GS** |
| Minimal internal engineering, vendor SLA | **Radisys** |
| Maximum control over NFs and supply chain | **Open5GS or free5GC** |
| Fastest time-to-deploy with zero telecom expertise | **Radisys** |

---

## 2. Architecture Deep Dive

### 2.1 Open5GS

```
┌─────────────────────────────────────────────────┐
│                   Open5GS Core                  │
│                                                 │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐     │
│  │ AMF │ │ SMF │ │ UPF │ │ NRF │ │ UDM │     │
│  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘     │
│     │       │       │       │       │          │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐     │
│  │ UDR │ │AUSF │ │ PCF │ │ NSSF│ │ SCP │     │
│  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘     │
│                                                 │
│  Also supports: EPC (MME, SGW, PGW, HSS, PCRF) │
└─────────────────────────────────────────────────┘
```

**Key Characteristics:**
- Implements **both EPC (4G) and 5GC** in a single codebase — unique advantage for hybrid deployments
- Monolithic-leaning design: fewer moving parts, each NF is a lightweight C binary
- NFs communicate via SBI (HTTP/2) for 5GC and GTP for EPC
- Single-process NFs (not microservice-split internally)
- Configuration via YAML files, straightforward to template

**Deployment Model:**
- Works well on **bare metal, VMs, and containers** (not opinionated about orchestration)
- Docker Compose and Helm charts available from community
- Can run all NFs on a **single node** for lab/dev scenarios
- Typically 5-8 containers for a full 5G SA core

**Strengths:**
- Lowest resource footprint of any open-source 5GC
- Hybrid 4G/5G in one stack — no separate EPC needed
- Mature codebase (~5,000+ commits, active since 2017 as NextEPC)
- Direct 3GPP spec mapping in C — easy to trace spec compliance
- Active maintainer (Sukchan Lee) with consistent release cadence

**Weaknesses:**
- HA and horizontal scaling require **custom engineering** (no built-in clustering)
- C codebase raises the bar for contributors (memory management, debugging)
- Less cloud-native patterning (no native service mesh integration)
- Community-driven governance — bus factor risk on lead maintainer

---

### 2.2 free5GC

```
┌──────────────────────────────────────────────────────────────┐
│                      free5GC Core (SBA)                      │
│                                                              │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ │
│  │ AMF │ │ SMF │ │ UPF │ │ NRF │ │ UDM │ │ UDR │ │AUSF │ │
│  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌──────┐ ┌──────┐ ┌─────┐       │
│  │ PCF │ │NSSF │ │ NEF │ │N3IWF │ │TNGF  │ │ CHF │       │
│  └─────┘ └─────┘ └─────┘ └──────┘ └──────┘ └─────┘       │
│                                                              │
│  + WebUI for subscriber management                           │
│  + gtp5g kernel module for UPF dataplane                     │
└──────────────────────────────────────────────────────────────┘
```

**Key Characteristics:**
- Strict **Service-Based Architecture (SBA)** per 3GPP spec
- Each NF is a separate Go binary with its own container
- More NFs implemented than Open5GS (N3IWF, TNGF, CHF, NEF)
- UPF uses a custom **gtp5g kernel module** (requires kernel-level access)
- WebUI included for subscriber provisioning

**Deployment Model:**
- Designed for **Kubernetes-first** deployment
- Official Docker Compose and Helm charts maintained
- 10-14 containers for a full deployment
- Requires **gtp5g kernel module** on UPF host (kernel 5.x+ recommended)
- UERANSIM integration well-documented for testing

**Strengths:**
- True cloud-native architecture — each NF independently scalable
- Go language — memory-safe, easier hiring, better tooling ecosystem
- Apache 2.0 license — most permissive, no copyleft concerns
- Academic backing (NCTU/NYCU) with industry collaboration
- Clean module separation — can replace individual NFs
- Built-in WebUI for subscriber management

**Weaknesses:**
- More containers = more orchestration complexity and resource overhead
- **gtp5g kernel module** is a hard dependency for UPF (limits where UPF can run)
- Multi-repo structure — tracking changes across repos is harder
- Some 3GPP edge cases may lag behind Open5GS
- Higher baseline resource consumption

---

### 2.3 Radisys 5G Core

```
┌──────────────────────────────────────────────────┐
│              Radisys 5G Core (Vendor)             │
│                                                   │
│  ┌───────────────────────────────────────────┐   │
│  │        Vendor-Packaged NF Bundle          │   │
│  │  AMF | SMF | UPF | NRF | UDM | AUSF | PCF│   │
│  │  (Pre-built, pre-configured images)       │   │
│  └───────────────────────────────────────────┘   │
│                                                   │
│  + Radisys Connect RAN integration               │
│  + Management & Orchestration portal             │
│  + Vendor support & SLA                          │
└──────────────────────────────────────────────────┘
```

**Key Characteristics:**
- Proprietary, vendor-packaged NF images
- Often bundled with Radisys RAN (Connect platform)
- Delivered as pre-built container images or VM appliances
- Management portal included

**Deployment Model:**
- Vendor-guided installation
- Typically requires vendor professional services for initial setup
- Can run on Kubernetes or VM infrastructure
- Vendor controls image lifecycle and updates

**Strengths:**
- Turnkey solution — minimal telecom expertise needed internally
- Vendor SLA and support contract
- Potentially pre-certified for specific use cases
- Integrated RAN+Core story if using Radisys RAN
- Vendor handles security patches and upgrades

**Weaknesses:**
- **Black-box**: no source code visibility, cannot audit or patch
- **Vendor lock-in**: switching cost is high, contractual exit
- Limited customization — must request features through vendor
- Cost: license fees + annual support + professional services
- Image opacity — cannot run SBOM analysis or slim images
- Dependency on vendor's release cadence for bug fixes

---

## 3. Network Functions Coverage

| Network Function | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **AMF** (Access & Mobility) | Yes | Yes | Yes |
| **SMF** (Session Management) | Yes | Yes | Yes |
| **UPF** (User Plane) | Yes | Yes (gtp5g kernel module) | Yes |
| **NRF** (NF Repository) | Yes | Yes | Yes |
| **UDM** (Unified Data Mgmt) | Yes | Yes | Yes |
| **UDR** (Unified Data Repo) | Yes | Yes | Yes |
| **AUSF** (Authentication) | Yes | Yes | Yes |
| **PCF** (Policy Control) | Yes | Yes | Yes |
| **NSSF** (Network Slice) | Yes | Yes | Vendor claim |
| **NEF** (Network Exposure) | Partial | Yes | Vendor claim |
| **N3IWF** (Non-3GPP Access) | No (community WIP) | Yes | Vendor claim |
| **TNGF** (Trusted Non-3GPP) | No | Yes | Unknown |
| **CHF** (Charging) | No | Yes | Vendor claim |
| **SCP** (Service Comm Proxy) | Yes | Partial | Unknown |
| **EPC (4G)** | **Yes (full)** | No | Separate product |
| **WebUI** | Community tools | **Yes (built-in)** | Yes (vendor portal) |

**Key Takeaway:** free5GC has the broadest NF coverage. Open5GS uniquely supports hybrid 4G/5G. Radisys claims are unverifiable without access.

---

## 4. Code Quality & Language Analysis

### 4.1 Open5GS (C)

| Aspect | Assessment |
|---|---|
| **Memory Safety** | Manual management — risk of buffer overflows, use-after-free |
| **Performance** | Excellent — deterministic, low-latency, minimal GC pauses |
| **Testability** | Unit tests exist, but fewer modern testing frameworks available for C |
| **Code Structure** | Well-organized, spec-traceable, consistent coding style |
| **Build System** | Meson + Ninja — fast, reliable |
| **Dependencies** | Minimal external deps (talloc, mongoc, yaml, nghttp2) |
| **Hiring Difficulty** | High — need C + telecom expertise intersection |
| **Debugging** | GDB, Valgrind, AddressSanitizer — powerful but steep learning curve |

### 4.2 free5GC (Go)

| Aspect | Assessment |
|---|---|
| **Memory Safety** | Garbage collected — no manual memory bugs |
| **Performance** | Good — slight overhead from GC, but adequate for private 5G |
| **Testability** | Go's built-in test framework, easy to write unit/integration tests |
| **Code Structure** | Clean separation per NF, standard Go project layout |
| **Build System** | Standard Go modules — simple, reproducible |
| **Dependencies** | More external deps (gin, mongo-driver, etc.) — larger attack surface |
| **Hiring Difficulty** | Low-Medium — large Go developer pool |
| **Debugging** | Delve debugger, pprof profiling — modern and accessible |

### 4.3 Radisys (Proprietary)

| Aspect | Assessment |
|---|---|
| **Memory Safety** | Unknown — cannot audit |
| **Performance** | Vendor-benchmarked (claims unverifiable independently) |
| **Testability** | Internal QA only — no visibility |
| **Code Structure** | N/A |
| **Build System** | N/A |
| **Dependencies** | Unknown |
| **Hiring Difficulty** | N/A — vendor handles |
| **Debugging** | Vendor support tickets only |

---

## 5. Container Footprint & Resource Requirements

| Metric | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **NF Container Count** | 5-8 | 10-14 | Vendor-defined |
| **Typical NF Image Size** | 30-80 MB (Alpine-based) | 80-200 MB (Go binaries) | 200 MB+ (opaque) |
| **Total Core Footprint** | ~300-600 MB | ~1-3 GB | Unknown (typically larger) |
| **Min RAM (lab)** | 2 GB | 4 GB | 8 GB+ |
| **Min CPU (lab)** | 2 cores | 4 cores | 4+ cores |
| **Can Slim Base Image** | Yes | Yes | No |
| **Can Build from Source** | Yes | Yes | No |
| **SBOM Generation** | Full control | Full control | Limited/None |
| **CVE Patch Control** | Self-managed | Self-managed | Vendor-dependent |
| **Custom Image Registry** | Yes | Yes | Vendor-specific |

### Image Optimization Potential

| Technique | Open5GS | free5GC | Radisys |
|---|---|---|---|
| Multi-stage builds | Yes | Yes | N/A |
| Alpine/distroless base | Yes | Yes | N/A |
| Strip debug symbols | Yes | Yes (Go -ldflags) | N/A |
| Remove unused NFs | Yes | Yes | N/A |
| Pin exact dependency versions | Yes | Yes (go.sum) | N/A |

---

## 6. Open Source Health & Community

### 6.1 GitHub Metrics (as of early 2026)

| Metric | Open5GS | free5GC |
|---|---|---|
| **GitHub Stars** | ~2,800+ | ~2,400+ |
| **Forks** | ~1,000+ | ~900+ |
| **Total Commits** | ~5,000+ | ~500+ (main repo) |
| **Contributors** | ~120+ | ~50+ |
| **Active Since** | 2017 (as NextEPC) | 2019 |
| **Release Cadence** | Regular (monthly-ish) | Periodic |
| **Issue Response Time** | Days | Days-Weeks |
| **PR Review Process** | Maintainer-reviewed | Team-reviewed |
| **CI/CD** | GitHub Actions | GitHub Actions |
| **Documentation** | Good (open5gs.org) | Good (free5gc.org) |

### 6.2 Community Health Signals

| Signal | Open5GS | free5GC |
|---|---|---|
| Organic commit history | Yes — consistent, real dev patterns | Yes — academic + community |
| Real contributor diversity | Yes — global contributors | Yes — NCTU/NYCU + external |
| Active issues & PR reviews | Yes | Yes |
| Release discipline | Yes — tagged releases with changelogs | Yes — tagged releases |
| Signs of star-botting | No evidence | No evidence |
| Mailing list / Discord / Forum | Active community channels | Active community channels |
| Conference presence | FOSDEM, telecom meetups | Academic papers, 5G summits |

**Verdict:** Both are healthy, legitimate open-source projects with real communities. Open5GS has deeper commit history; free5GC has stronger academic backing.

---

## 7. 3GPP Standards Compliance

| Aspect | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Target 3GPP Release** | Release 17 (some Rel-18 items) | Release 15/16 | Vendor-claimed Rel-16+ |
| **Spec Traceability** | Direct C implementation maps to TS specs | SBA mapping to TS specs | Opaque |
| **SBI Compliance** | HTTP/2 based SBI | HTTP/2 based SBI | Vendor implementation |
| **NAS Protocol** | Full 5G NAS | Full 5G NAS | Vendor |
| **NGAP** | Full | Full | Vendor |
| **GTP-U** | Native | gtp5g kernel module | Vendor |
| **Network Slicing** | Basic NSSF | NSSF + NEF | Vendor claim |
| **Interop Testing** | Community (UERANSIM, srsRAN) | Community (UERANSIM, srsRAN) | Vendor lab |
| **Certification** | No formal cert | No formal cert | Possible vendor cert |

**Key Insight:**
- If you need **formal certification documentation** → Radisys has advantage (vendor provides it)
- If you need **spec traceability and auditability** → Open5GS or free5GC (you can read the code)
- Open5GS tracks higher 3GPP releases, free5GC has broader NF coverage

---

## 8. Operational Considerations (Day-2)

### 8.1 Deployment & Configuration

| Aspect | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Config Format** | YAML | YAML | Vendor portal/config |
| **Config Complexity** | Low-Medium | Medium-High (more NFs) | Low (vendor handles) |
| **Helm Charts** | Community | Official + community | Vendor |
| **Subscriber Mgmt** | MongoDB + scripts | WebUI + MongoDB | Vendor portal |
| **TLS/mTLS** | Configurable | Configurable | Vendor-managed |

### 8.2 Monitoring & Observability

| Aspect | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Prometheus Metrics** | Basic (community contribution) | Basic (community) | Vendor-specific |
| **Log Format** | Structured logs | Structured logs | Vendor format |
| **Tracing (OpenTelemetry)** | Community effort | Community effort | Vendor |
| **Health Checks** | Process-level | HTTP endpoints | Vendor |
| **Alerting** | DIY (Prometheus/Grafana) | DIY (Prometheus/Grafana) | Vendor portal |

### 8.3 Upgrade & Rollback

| Aspect | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Upgrade Complexity** | Low-Moderate | Higher (more NFs to coordinate) | Vendor-managed |
| **Rolling Upgrades** | Manual orchestration | K8s-native possible | Vendor procedure |
| **Rollback** | Container image pinning | Container image pinning | Vendor procedure |
| **DB Migration** | Manual (MongoDB schema) | Manual (MongoDB schema) | Vendor handles |
| **Breaking Changes** | Documented in release notes | Documented in release notes | Vendor communication |

### 8.4 Scaling & HA

| Aspect | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Horizontal Scaling** | Custom engineering needed | K8s HPA possible per NF | Vendor-defined |
| **HA / Failover** | No built-in (needs external) | No built-in (needs external) | Vendor may provide |
| **Load Balancing** | External LB needed | External LB / SCP | Vendor |
| **Session Persistence** | Application-level | Application-level | Vendor |
| **Geographic Redundancy** | DIY | DIY | Vendor option |

---

## 9. Security & Supply Chain

| Control | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **Build from source** | Yes | Yes | No |
| **Sign your own images** | Yes | Yes | No |
| **Pin dependencies** | Yes (meson.build) | Yes (go.sum) | Vendor-dependent |
| **Remove unnecessary libs** | Yes | Yes | No |
| **SBOM generation** | Full (Syft, Trivy, etc.) | Full | Limited |
| **CVE scanning** | Full (Trivy, Grype) | Full | Vendor responsibility |
| **Patch turnaround** | Self-controlled | Self-controlled | Vendor SLA |
| **License compliance** | AGPLv3 (copyleft) | Apache 2.0 (permissive) | Proprietary |
| **Audit capability** | Full source audit | Full source audit | None |

### License Implications

| Scenario | Open5GS (AGPLv3) | free5GC (Apache 2.0) | Radisys |
|---|---|---|---|
| Internal use only | Safe | Safe | Per contract |
| Modifications shared as service | **Must share source** | No obligation | N/A |
| Embedding in commercial product | **Copyleft applies** | Permissive — OK | Per contract |
| Forking for proprietary use | **Not allowed** | Allowed | N/A |

**Important:** If you plan to offer a 5G-based service commercially, free5GC's Apache 2.0 license is significantly more permissive than Open5GS's AGPLv3.

---

## 10. Cost Analysis

### 10.1 Open5GS — Total Cost of Ownership

| Cost Item | Estimate |
|---|---|
| Software License | **$0** |
| Infrastructure (3-node K8s lab) | $200-500/mo cloud or one-time HW |
| Engineering (setup + customize) | 2-4 weeks senior telecom engineer |
| Ongoing Maintenance | 0.25-0.5 FTE |
| Support | Community only (no SLA) |
| **Year-1 Estimate** | **$50K-100K** (mostly engineering time) |

### 10.2 free5GC — Total Cost of Ownership

| Cost Item | Estimate |
|---|---|
| Software License | **$0** |
| Infrastructure (5-node K8s cluster) | $300-800/mo cloud or one-time HW |
| Engineering (setup + customize) | 2-4 weeks Go/K8s engineer |
| Ongoing Maintenance | 0.25-0.5 FTE |
| Support | Community only (no SLA) |
| **Year-1 Estimate** | **$60K-120K** (more infra, easier hiring) |

### 10.3 Radisys — Total Cost of Ownership

| Cost Item | Estimate |
|---|---|
| Software License | **$100K-500K+/yr** (varies by scale) |
| Infrastructure | Vendor-specified HW/cloud |
| Professional Services (setup) | $20K-50K+ |
| Annual Support & Maintenance | 15-20% of license |
| Engineering (integration) | 1-2 weeks |
| **Year-1 Estimate** | **$200K-600K+** |

---

## 11. Testing & Simulation Ecosystem

| Tool | Open5GS | free5GC | Radisys |
|---|---|---|---|
| **UERANSIM** | Well-tested integration | Well-tested integration | Possible (vendor confirms) |
| **srsRAN** | Community tested | Community tested | Vendor lab |
| **PacketRusher** | Compatible | Compatible | Unknown |
| **gNBSim** | Compatible | Compatible | Unknown |
| **Commercial UE** | Tested with various | Tested with various | Vendor-validated |

---

## 12. Migration & Interoperability

### Migrating Between Options

| Migration Path | Difficulty | Notes |
|---|---|---|
| Open5GS → free5GC | Medium | Both use MongoDB; subscriber data portable. NF configs differ. |
| free5GC → Open5GS | Medium | Reverse of above. Lose NFs not in Open5GS (N3IWF, CHF). |
| Either OSS → Radisys | High | Full re-provisioning. No config portability. |
| Radisys → Either OSS | High | Need to extract subscriber data. Vendor cooperation needed. |

### RAN Interoperability

| RAN Solution | Open5GS | free5GC | Radisys |
|---|---|---|---|
| srsRAN | Tested | Tested | Unknown |
| UERANSIM (sim) | Tested | Tested | Tested |
| Nokia | Community reports | Community reports | Likely tested |
| Ericsson | Community reports | Community reports | Likely tested |
| Radisys RAN | Should work (3GPP) | Should work (3GPP) | **Native integration** |
| Baicells | Community tested | Community tested | Unknown |

---

## 13. Recommended Decision Framework

### Choose Open5GS if:
- VM-based or bare-metal deployment preferred
- Hybrid **LTE + 5G** requirement (unique advantage)
- Small team with C/telecom expertise
- Minimal container sprawl desired
- Resource-constrained environment (edge, IoT gateway)
- Need the lowest possible footprint

### Choose free5GC if:
- **Kubernetes-native** deployment is the target
- Team has Go and cloud-native expertise
- Long-term customization and NF-level scaling expected
- **Apache 2.0 license** needed (commercial product embedding)
- Need broadest NF coverage (N3IWF, CHF, NEF, TNGF)
- DevOps maturity exists in the organization

### Choose Radisys if:
- Internal telecom/5G expertise is **limited or absent**
- **Vendor SLA and support** are contractual requirements
- Budget allows for license + support costs
- Integrated RAN+Core from a single vendor is preferred
- Time-to-deploy is the primary constraint
- Willing to accept image opacity and vendor dependency

---

## 14. Risk Register

| Risk | Open5GS | free5GC | Radisys | Mitigation |
|---|---|---|---|---|
| **Lead maintainer leaves** | HIGH (single key maintainer) | Medium (team/academic) | Low (vendor) | Fork + community; Diversify contributors |
| **Security vulnerability** | Self-patch (fast) | Self-patch (fast) | Vendor SLA (slow) | OSS: immediate patching. Vendor: escalation SLA. |
| **Feature gap** | Build it yourself | Build it yourself | Request from vendor | OSS: contribute/fork. Vendor: contract negotiation. |
| **Vendor bankruptcy** | N/A | N/A | HIGH | Fork OSS alternative as backup plan |
| **License dispute** | Medium (AGPLv3 complex) | Low (Apache 2.0 simple) | Per contract | Legal review of AGPLv3 obligations |
| **Performance ceiling** | Medium (optimize C) | Medium (GC tuning) | Unknown | Load test before production |
| **Upgrade breaks** | Medium | Higher (more NFs) | Low (vendor tested) | Staging environment, canary deployments |

---

## 15. Governance Checklist (Pre-Decision)

Before making a final decision, verify the following:

- [ ] Evaluate release cadence over the last 12 months
- [ ] Verify CVE handling and response time
- [ ] Check number of active maintainers (bus factor)
- [ ] Validate performance under your expected subscriber load
- [ ] Test upgrade and rollback strategy in a staging environment
- [ ] Verify SBOM availability (especially for vendor option)
- [ ] Confirm support for required NFs (N3IWF, slicing, VoNR, etc.)
- [ ] Review license compatibility with your business model
- [ ] Assess team skillset alignment (C vs Go vs vendor-managed)
- [ ] Calculate 3-year TCO including engineering time
- [ ] Test RAN interoperability with your chosen gNB
- [ ] Evaluate monitoring/alerting integration with your existing stack
- [ ] Document data migration path if switching later
- [ ] Verify kernel compatibility for free5GC's gtp5g module (if choosing free5GC)

---

## 16. Proof of Concept Recommendations

### Minimum Viable PoC Setup

| Component | Open5GS PoC | free5GC PoC |
|---|---|---|
| **Infra** | 1 VM (4 CPU, 8 GB RAM) | 3-node K8s (4 CPU, 8 GB each) |
| **OS** | Ubuntu 22.04+ | Ubuntu 22.04+ (kernel 5.x+) |
| **Simulator** | UERANSIM | UERANSIM |
| **Duration** | 1-2 weeks | 2-3 weeks |
| **Test Cases** | Registration, PDU session, handover | Registration, PDU session, slicing |
| **Success Criteria** | UE attach, data plane, basic mobility | UE attach, data plane, NF scaling |

### PoC Evaluation Scorecard

| Criterion | Weight | Score (1-5) | Notes |
|---|---|---|---|
| Time to first UE attach | 15% | | |
| Stability over 48h run | 20% | | |
| Resource consumption | 15% | | |
| Configuration complexity | 10% | | |
| Documentation quality | 10% | | |
| Team comfort level | 15% | | |
| NF coverage for use case | 15% | | |
| **Total** | **100%** | | |

---

*This document should be reviewed and updated quarterly as the 5G open-source ecosystem evolves rapidly.*
