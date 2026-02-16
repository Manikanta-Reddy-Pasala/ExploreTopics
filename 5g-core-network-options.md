Below is a ready-to-paste Confluence page guide comparing:
• Open5GS
• free5GC
• Radisys 5G Core

Assumption:
👉 Private 5G / Lab / Enterprise deployment (NOT Tier-1 carrier grade)
👉 Focus on architecture control, OSS health, image footprint, code quality, standards discipline, and long-term risk.

⸻

📘 5G Core Evaluation Guide

Scope: Architecture Risk, Code Quality, Image Footprint, Governance & Maintainability

⸻

1️⃣ Executive Summary

Area Open5GS free5GC Radisys
License AGPLv3 Apache 2.0 Proprietary
Language C Go Vendor internal
Container Footprint Low–Medium Medium–High High
OSS Governance Community Linux Foundation style Vendor
Modify & Rebuild Yes Yes No
Supply Chain Control High High Low
Exit Risk Low Low Medium–High
Complexity Moderate Higher Hidden complexity

Quick Recommendation (Private 5G)
• Kubernetes-native → free5GC
• VM-based lab / simpler core → Open5GS
• Minimal internal engineering effort → Radisys

⸻

2️⃣ Architecture Comparison

Open5GS Architecture

Characteristics
• Implements EPC + 5GC
• Fewer moving parts
• Less microservice fragmentation
• Suitable for VM or small container setup

Risk Notes
• HA and scale require custom engineering
• Less cloud-native patterning

⸻

free5GC Architecture

Characteristics
• Strict Service-Based Architecture (SBA)
• Separate NF containers (AMF, SMF, NRF, PCF, etc.)
• Designed for cloud-native environments

Risk Notes
• More containers → more DevOps overhead
• Higher integration complexity

⸻

Radisys 5G Core Architecture

Characteristics
• Vendor-packaged NFs
• Often integrated with RAN solutions
• Delivered as multi-component images

Risk Notes
• Black-box behavior
• Dependency on vendor for fixes
• Limited visibility into internals

⸻

3️⃣ Code Quality & Language Risk

Open5GS
• Language: C
• Manual memory management
• Performance efficient
• Higher risk of memory-related bugs if modified
• Requires telecom + C expertise

Strength
• Deterministic performance
• Telecom-style implementation close to 3GPP spec

Risk
• Harder onboarding
• Fewer modern testing frameworks

⸻

free5GC
• Language: Go
• Memory-safe (garbage collected)
• Easier to structure microservices
• Easier hiring pool

Strength
• Clean module separation
• Cloud-native friendly
• Easier CI/CD integration

Risk
• Microservice complexity
• Some features may lag edge-case specs

⸻

Radisys
• Proprietary implementation
• Internal QA unknown
• No code visibility

Risk
• Cannot audit or patch
• Cannot modify NFs
• Must trust vendor testing

⸻

4️⃣ Image Size & Container Footprint

Area Open5GS free5GC Radisys
NF Count ~5–8 ~8–12+ Vendor defined
Can slim base image Yes Yes No
SBOM control Full Full Limited
CVE patch control You You Vendor

Observations
• Open5GS → lowest footprint
• free5GC → higher orchestration load
• Radisys → opaque images, often larger footprint

⸻

5️⃣ Open Source Health & GitHub Governance

Open5GS
• ~2.5k stars
• ~4,800+ commits
• 100+ contributors
• Multi-year release history
• Human-reviewed PRs
• No bot-only patterns

Risk
• Community-driven (no formal foundation governance)

⸻

free5GC
• ~2.2k stars
• ~470+ commits (main repo)
• ~48 contributors
• Signed GitHub releases
• Governance repository exists

Risk
• Some work distributed across multiple repos
• Requires tracking multiple modules

⸻

Bot or Fake Risk Assessment

Indicator Open5GS free5GC
Organic commit history Yes Yes
Real contributor diversity Yes Yes
Active issues & PR reviews Yes Yes
Release discipline Yes Yes
Signs of star-botting No evidence No evidence

Conclusion: Both are healthy, real OSS projects.

⸻

6️⃣ Standards & 3GPP Discipline

Area Open5GS free5GC Radisys
3GPP Alignment Release 17 claimed Release 15+ Vendor claim
Spec Mapping Direct C mapping SBA mapping Opaque
Interop Testing Community driven Community driven Likely vendor lab tested

If certification documentation required → vendor advantage.
If flexibility required → OSS advantage.

⸻

7️⃣ Operational Risk (Day-2)

Upgrade Risk
• Open5GS: moderate
• free5GC: higher (more services)
• Radisys: vendor controlled

Performance Regression Risk

Both OSS projects show real-world fixes in release notes (e.g., UPF metric removal for performance in Open5GS).
This is a positive signal: issues acknowledged and resolved.

⸻

8️⃣ Supply Chain Risk

Control Open5GS free5GC Radisys
Build from source Yes Yes No
Sign your own images Yes Yes No
Pin dependencies Yes Yes Vendor dependent
Remove unnecessary libs Yes Yes No

For organizations concerned about binary opacity → OSS wins.

⸻

9️⃣ Long-Term Maintainability

Factor Open5GS free5GC Radisys
Team skill portability Medium High Low
Vendor dependency Low Low High
Custom feature ability High High Limited
Exit strategy Forkable Forkable Contractual


⸻

🔟 Final Risk Positioning (Non-Commercial Grade)

Lowest Architecture Complexity → Open5GS

Best Cloud-Native Modularity → free5GC

Least Engineering Effort → Radisys

Highest Control → Open5GS / free5GC

Highest Dependency → Radisys

⸻

📌 Recommended Decision Framework

Choose Open5GS if:
• VM-based deployment
• Hybrid LTE + 5G
• Small team
• Minimal container sprawl desired

Choose free5GC if:
• Kubernetes-native
• Microservice familiarity
• Long-term customization expected
• DevOps maturity exists

Choose Radisys if:
• Internal telecom expertise limited
• Vendor SLA required
• Willing to accept image opacity

⸻

📎 Governance Checklist (Add This to Confluence)

Before final decision:
• Evaluate release cadence over last 12 months
• Verify CVE handling response time
• Check number of active maintainers
• Validate performance under expected load
• Test upgrade rollback strategy
• Verify SBOM availability (especially vendor case)
• Confirm support for required NFs (N3IWF, slicing, VoNR, etc.)

⸻
