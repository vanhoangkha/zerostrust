# Part 7: Lộ trình triển khai Zero Trust cho doanh nghiệp

## Giới thiệu

Qua 6 phần trước, chúng ta đã phân tích Zero Trust từ mọi góc độ: nguyên lý nền tảng (Part 1), kiến trúc chuẩn NIST (Part 2), identity và micro-segmentation (Part 3), network access hiện đại (Part 4), cloud-native và Kubernetes (Part 5), và AI workloads (Part 6).

Câu hỏi còn lại: **Bắt đầu từ đâu? Triển khai theo thứ tự nào? Mất bao lâu? Chi phí ra sao?**

Part 7 này tổng hợp tất cả thành một **lộ trình triển khai thực tế** — từ assessment đến production, từ quick wins đến full maturity. Đây không phải lý thuyết — đây là playbook dựa trên kinh nghiệm triển khai thực tế và CISA Zero Trust Maturity Model v2.

## Trước khi bắt đầu: Đánh giá hiện trạng

### CISA Zero Trust Maturity Model v2 — 5 Mức độ trưởng thành

| Mức | Tên | Đặc điểm | % Doanh nghiệp 2026 |
|-----|-----|-----------|---------------------|
| 0 | Traditional | Perimeter-based, implicit trust | ~15% |
| 1 | Initial | Bắt đầu nhận diện gaps, pilot projects | ~30% |
| 2 | Advanced | Consistent policy enforcement, automation emerging | ~35% |
| 3 | Optimal | Dynamic policy, continuous verification, full visibility | ~15% |
| 4 | Strategic | AI-driven, adaptive, predictive security | ~5% |

### 5 Pillars đánh giá (theo CISA)

```
┌─────────────────────────────────────────────────────────────────┐
│                    ZERO TRUST MATURITY                           │
│                                                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────┐ │
│  │ Identity │ │ Devices  │ │ Networks │ │ Apps &   │ │ Data │ │
│  │          │ │          │ │          │ │ Workloads│ │      │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────┘ │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              Cross-cutting: Visibility & Analytics           ││
│  │              Cross-cutting: Automation & Orchestration       ││
│  │              Cross-cutting: Governance                       ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Assessment Checklist

| Pillar | Câu hỏi đánh giá | Red Flag |
|--------|-------------------|----------|
| **Identity** | MFA adoption rate? Passwordless %? NHI inventory? | MFA < 80%, no NHI tracking |
| **Devices** | EDR coverage? Device posture check trước access? | No posture-based access decisions |
| **Networks** | Flat network? East-west traffic encrypted? | Single VLAN, no segmentation |
| **Apps/Workloads** | Shadow IT inventory? WAF/API security? | Unknown apps > 20% |
| **Data** | Data classification? DLP deployed? Encryption at rest? | No data classification |

## Lộ trình 4 giai đoạn

### Tổng quan Timeline

```
┌────────────────┬────────────────┬────────────────┬────────────────┐
│   Phase 1      │   Phase 2      │   Phase 3      │   Phase 4      │
│   Foundation   │   Core         │   Advanced     │   Optimization │
│                │   Controls     │   Capabilities │                │
│   Month 1-3   │   Month 4-9    │   Month 10-18  │   Month 18+    │
│                │                │                │   (Continuous)  │
├────────────────┼────────────────┼────────────────┼────────────────┤
│ • Assessment   │ • Identity     │ • Micro-seg    │ • AI-driven    │
│ • Quick wins   │   platform     │ • ZTNA/SASE    │   policy       │
│ • Governance   │ • Device trust │ • Cloud-native │ • Predictive   │
│ • Architecture │ • Data         │ • AI workload  │   security     │
│   design       │   classification│  security     │ • Continuous   │
│                │ • Initial seg  │ • Advanced     │   optimization │
│                │                │   monitoring   │                │
└────────────────┴────────────────┴────────────────┴────────────────┘
```

---

## Phase 1: Foundation (Tháng 1-3)

### Mục tiêu
Xây dựng nền tảng: governance, visibility, và quick wins để tạo momentum.

### 1.1 Governance & Stakeholder Buy-in

| Hoạt động | Output | Owner |
|-----------|--------|-------|
| Thành lập Zero Trust Steering Committee | Charter document | CISO |
| Xác định Executive Sponsor | Budget approval | CIO/CEO |
| Định nghĩa scope và priority systems | Protect Surface list | Security Architect |
| Xây dựng business case + ROI model | Board presentation | CISO + CFO |

**Critical Success Factor:** Zero Trust là **business transformation**, không chỉ security project. Cần buy-in từ C-suite.

### 1.2 Discovery & Assessment

```
Asset Discovery
├── Identity inventory
│   ├── Human identities (workforce, contractors, partners)
│   ├── Non-human identities (service accounts, API keys, AI agents)
│   └── Privileged accounts
├── Device inventory
│   ├── Corporate-managed devices
│   ├── BYOD
│   └── IoT/OT devices
├── Application inventory
│   ├── SaaS applications
│   ├── Internal applications
│   ├── APIs
│   └── AI/ML workloads
├── Data inventory
│   ├── Data classification (Confidential/Internal/Public)
│   ├── Data flow mapping
│   └── Crown jewels identification
└── Network topology
    ├── Current segmentation
    ├── East-west traffic flows
    └── External connections
```

### 1.3 Define Protect Surfaces (theo John Kindervag)

Thay vì cố bảo vệ toàn bộ attack surface (vô hạn), Zero Trust tập trung vào **Protect Surfaces** — những thứ quan trọng nhất:

| DAAS Category | Ví dụ | Priority |
|---------------|-------|----------|
| **D**ata | Customer PII, financial data, IP | Critical |
| **A**pplications | ERP, CRM, payment processing | High |
| **A**ssets | Domain controllers, HSMs, AI models | High |
| **S**ervices | DNS, DHCP, Active Directory | Critical |

**Bắt đầu với 1-2 protect surfaces nhỏ → chứng minh giá trị → mở rộng.**

### 1.4 Quick Wins (ROI ngay lập tức)

| Quick Win | Effort | Impact | Timeline |
|-----------|--------|--------|----------|
| Enable MFA cho tất cả admin accounts | Low | Critical | Week 1-2 |
| Disable legacy protocols (NTLM, TLS 1.0/1.1) | Medium | High | Week 2-4 |
| Inventory và rotate stale service account credentials | Medium | High | Week 3-6 |
| Implement conditional access policies (location, device) | Medium | High | Week 4-8 |
| Deploy DNS filtering (block known malicious domains) | Low | Medium | Week 1-2 |
| Enable audit logging cho critical systems | Low | High | Week 1-3 |

### 1.5 Architecture Design

Output của Phase 1: **Zero Trust Architecture Design Document** bao gồm:

1. Target architecture diagram (PDP/PEP placement)
2. Technology stack selection
3. Integration points với existing infrastructure
4. Migration strategy (phased, không big-bang)
5. Success metrics và KPIs

---

## Phase 2: Core Controls (Tháng 4-9)

### Mục tiêu
Triển khai core pillars: Identity platform, device trust, data protection, initial network segmentation.

### 2.1 Identity Platform (Trụ cột #1)

| Component | Implementation | Technology Examples |
|-----------|---------------|---------------------|
| **Identity Provider (IdP)** | Centralized, cloud-native IdP | Entra ID, Okta, Ping Identity |
| **Phishing-resistant MFA** | FIDO2/Passkeys cho toàn bộ workforce | YubiKey, Windows Hello, Passkeys |
| **Passwordless** | Eliminate passwords cho user-facing auth | FIDO2 + Certificate-based |
| **SSO** | Single pane cho tất cả applications | SAML 2.0, OIDC |
| **Privileged Access Management** | Just-in-Time, session recording | CyberArk, BeyondTrust, Entra PIM |
| **NHI Management** | Workload identity, short-lived tokens | SPIFFE/SPIRE, Managed Identities |
| **Identity Governance** | Access reviews, lifecycle management | SailPoint, Saviynt |

#### Identity Maturity Progression

```
Level 1: Passwords + SMS MFA
    ↓
Level 2: Phishing-resistant MFA (FIDO2) + Conditional Access
    ↓
Level 3: Passwordless + Risk-based Authentication + PAM
    ↓
Level 4: Continuous Identity Verification + Behavioral Analytics
    ↓
Level 5: AI-driven Identity Decisions + Predictive Risk Scoring
```

### 2.2 Device Trust (Trụ cột #2)

| Capability | Requirement | Tool Examples |
|------------|-------------|---------------|
| Device registration | All devices enrolled in MDM/UEM | Intune, Jamf, SOTI |
| Health attestation | OS patched, AV running, encryption on | Compliance policies |
| Posture-based access | Non-compliant device = limited access | Conditional Access + EDR |
| EDR/XDR | Real-time threat detection on endpoint | CrowdStrike, Defender, SentinelOne |
| Certificate-based identity | Device identity via PKI | SCEP/NDES certificates |

#### Decision Flow: Device Trust

```
Device requests access
        │
        ▼
┌─────────────────┐     NO      ┌──────────────────┐
│ Device enrolled? │────────────▶│ Redirect to       │
│ (MDM/UEM)       │             │ enrollment portal │
└────────┬────────┘             └──────────────────┘
         │ YES
         ▼
┌─────────────────┐     NO      ┌──────────────────┐
│ Compliant?      │────────────▶│ Limited access    │
│ (patched, AV,   │             │ (web only, no    │
│  encrypted)     │             │  sensitive data)  │
└────────┬────────┘             └──────────────────┘
         │ YES
         ▼
┌─────────────────┐     HIGH    ┌──────────────────┐
│ Risk score?     │────────────▶│ Step-up auth +   │
│ (EDR signals)   │             │ session limits   │
└────────┬────────┘             └──────────────────┘
         │ LOW
         ▼
┌─────────────────┐
│ Full access     │
│ (per policy)    │
└─────────────────┘
```

### 2.3 Data Protection (Trụ cột #3)

| Stage | Action | Outcome |
|-------|--------|---------|
| **Classify** | Auto-classify data (sensitivity labels) | Mọi document/database có label |
| **Protect** | Encryption at rest + in transit + in use | Data unreadable nếu unauthorized |
| **Monitor** | DLP policies cho email, cloud, endpoint | Alert/block sensitive data leakage |
| **Control** | Rights management (IRM) | Data tự bảo vệ dù ở đâu |
| **Govern** | Retention, disposal policies | Minimize data exposure surface |

### 2.4 Initial Network Segmentation

Bắt đầu với **macro-segmentation** trước khi tiến tới micro-segmentation:

| Zone | Mục đích | Traffic Rules |
|------|----------|---------------|
| **Internet Zone** | Public-facing services | WAF, DDoS protection, API gateway |
| **User Zone** | Workforce access | Identity-verified, device-compliant only |
| **Application Zone** | Business apps, APIs | Service mesh, mTLS between services |
| **Data Zone** | Databases, storage | Strict access control, encryption, audit |
| **Management Zone** | Admin tools, jump servers | PAM-gated, session recorded |
| **AI/ML Zone** | Models, training, inference | Isolated compute, NHI controls |

---

## Phase 3: Advanced Capabilities (Tháng 10-18)

### Mục tiêu
Triển khai advanced controls: ZTNA/SASE, micro-segmentation, cloud-native security, AI workload protection.

### 3.1 ZTNA & SASE Deployment

| Step | Action | Business Outcome |
|------|--------|-----------------|
| 1 | Deploy ZTNA cho remote workforce | VPN replacement, better UX |
| 2 | Migrate web filtering to SWG | Consistent policy mọi nơi |
| 3 | Enable CASB cho SaaS apps | Shadow IT visibility + control |
| 4 | Integrate DLP across channels | Unified data protection |
| 5 | Enable Digital Experience Monitoring | Troubleshoot user issues proactively |
| 6 | Full SASE consolidation | Single policy engine, reduced complexity |

#### ZTNA Migration Strategy (từ VPN)

```
Phase A: Coexistence
├── VPN vẫn hoạt động cho legacy apps
├── ZTNA deployed cho new/cloud apps
└── Users dùng cả hai (parallel)

Phase B: Primary ZTNA
├── ZTNA là default access method
├── VPN chỉ cho legacy apps không migrate được
└── Metrics: 80%+ traffic qua ZTNA

Phase C: VPN Decommission
├── Tất cả apps accessible qua ZTNA
├── VPN retired
└── Metrics: 100% ZTNA, zero VPN
```

### 3.2 Micro-segmentation at Scale

| Approach | Use Case | Technology |
|----------|----------|------------|
| **Network-based** | Traditional datacenter | NSX, Cisco ACI, Illumio |
| **Host-based** | Server workloads | Host firewalls, Guardicore |
| **Container-native** | Kubernetes | Cilium, Calico NetworkPolicy |
| **Identity-based** | Service-to-service | Service mesh (Istio, Linkerd) |
| **Application-layer** | API-level control | API gateways, WAF rules |

#### Implementation Order

1. **Map traffic flows** (baseline 2-4 weeks, tool-assisted)
2. **Label workloads** (environment, tier, sensitivity)
3. **Define policies** (deny-all default, allow-list per service)
4. **Deploy in monitor mode** (detect violations without blocking)
5. **Enforce gradually** (start with non-production, move to production)
6. **Automate** (policy-as-code, CI/CD integration)

### 3.3 Cloud-Native Zero Trust (Kubernetes)

| Layer | Control | Implementation |
|-------|---------|----------------|
| **Cluster** | API server hardening, RBAC | OPA/Gatekeeper, Kyverno |
| **Namespace** | Resource quotas, NetworkPolicy | Default-deny per namespace |
| **Pod** | Security context, service accounts | Pod Security Standards |
| **Service** | mTLS, authorization policies | Istio/Cilium service mesh |
| **Data** | Secrets management, encryption | Vault, Sealed Secrets, KMS |
| **Supply Chain** | Image signing, SBOM | Sigstore, Trivy, Grype |

### 3.4 AI Workload Security

| Control | Purpose | Reference |
|---------|---------|-----------|
| Agent Identity (AIT) | Per-agent identity lifecycle | Microsoft ZT4AI |
| Tool Allow-lists | Restrict agent capabilities | Principle of least privilege |
| Retrieval Authorization | Data-level access in RAG | Access control metadata |
| Behavioral Monitoring | Detect compromised agents | Baseline + anomaly detection |
| Output Sanitization | Prevent data leakage via responses | DLP at inference boundary |
| Human-in-the-Loop | Gate high-impact agent actions | Approval workflows |

### 3.5 Advanced Monitoring & Analytics

```
┌─────────────────────────────────────────────────────────────┐
│                  Security Operations Center                   │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │    SIEM     │  │    SOAR     │  │  Threat Intelligence│ │
│  │ Log aggreg. │  │ Automation  │  │  Feed integration   │ │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ │
│         │                │                     │            │
│         └────────────────┼─────────────────────┘            │
│                          ▼                                  │
│              ┌─────────────────────┐                        │
│              │   XDR Platform      │                        │
│              │   Correlation +     │                        │
│              │   AI-driven detect  │                        │
│              └─────────────────────┘                        │
│                          │                                  │
│         ┌────────────────┼────────────────┐                 │
│         ▼                ▼                ▼                 │
│  ┌────────────┐  ┌────────────┐  ┌────────────────┐        │
│  │ Identity   │  │ Network    │  │ Endpoint       │        │
│  │ Analytics  │  │ Analytics  │  │ Telemetry      │        │
│  │ (UEBA)     │  │ (NTA/NDR)  │  │ (EDR/XDR)      │        │
│  └────────────┘  └────────────┘  └────────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

---

## Phase 4: Optimization & Continuous Improvement (Tháng 18+)

### Mục tiêu
Đạt mức Optimal/Strategic trên CISA Maturity Model. AI-driven decisions, predictive security, continuous optimization.

### 4.1 AI-Driven Policy Engine

| Capability | Description | Benefit |
|------------|-------------|---------|
| Adaptive access policies | Risk score tự điều chỉnh theo behavior | Reduce friction cho low-risk, increase cho anomalies |
| Automated policy generation | ML đề xuất policies từ traffic patterns | 10x faster policy creation |
| Predictive risk scoring | Identify threats trước khi exploit | Proactive vs reactive |
| Automated response | SOAR playbooks trigger tự động | Mean-time-to-respond giảm 90% |

### 4.2 Metrics & KPIs

| Category | KPI | Target |
|----------|-----|--------|
| **Identity** | MFA adoption rate | 100% |
| **Identity** | Mean time to revoke access | < 1 hour |
| **Identity** | Passwordless adoption | > 80% |
| **Device** | Compliant device rate | > 95% |
| **Device** | Mean time to patch critical | < 24 hours |
| **Network** | % traffic với mTLS | > 90% |
| **Network** | East-west visibility coverage | 100% |
| **Data** | Data classification coverage | > 95% |
| **Data** | DLP incident response time | < 4 hours |
| **Apps** | Applications behind ZTNA | 100% |
| **Apps** | Vulnerability mean-time-to-remediate | < 7 days |
| **AI** | AI agent identity coverage | 100% |
| **AI** | Agent behavioral baseline coverage | > 90% |
| **Overall** | CISA Maturity Score | Level 3+ |

### 4.3 Continuous Improvement Cycle

```
        ┌──────────────┐
        │   MEASURE    │
        │  (Metrics,   │
        │   KPIs,      │
        │   Incidents) │
        └──────┬───────┘
               │
               ▼
┌──────────────────────────────┐
│          ANALYZE             │
│  (Gap analysis, threat      │
│   landscape changes,        │
│   technology evolution)     │
└──────────────┬───────────────┘
               │
               ▼
        ┌──────────────┐
        │    PLAN      │
        │  (Roadmap    │
        │   updates,   │
        │   new        │
        │   controls)  │
        └──────┬───────┘
               │
               ▼
        ┌──────────────┐
        │  IMPLEMENT   │
        │  (Deploy,    │
        │   configure, │
        │   test)      │
        └──────┬───────┘
               │
               └────────────────▶ (Back to MEASURE)
```

---

## Chi phí và ROI

### Ước tính chi phí theo quy mô doanh nghiệp

| Quy mô | Nhân sự | Ước tính chi phí (Year 1) | Ước tính chi phí (Ongoing/year) |
|---------|---------|---------------------------|----------------------------------|
| SMB (< 500 users) | 500 | $150K - $400K | $80K - $200K |
| Mid-market (500-5,000) | 2,500 | $500K - $2M | $300K - $800K |
| Enterprise (5,000-50,000) | 20,000 | $2M - $10M | $1M - $5M |
| Large Enterprise (50,000+) | 100,000 | $10M - $50M | $5M - $20M |

*Bao gồm: licenses, professional services, internal headcount, training. Không bao gồm: hardware refresh, network redesign nếu cần.*

### ROI Justification

| Metric | Before Zero Trust | After Zero Trust | Improvement |
|--------|-------------------|-----------------|-------------|
| Mean time to detect breach | 207 days | < 24 hours | 99%+ |
| Mean time to contain breach | 70 days | < 4 hours | 99%+ |
| Average breach cost | $4.45M (2025 avg) | Reduced 40-60% | $2-3M savings |
| Cyber insurance premium | Baseline | 15-25% reduction | Direct savings |
| Compliance audit findings | Multiple critical | Near-zero | Reduced risk |
| VPN infrastructure cost | $X/year | Eliminated | Cost saving |
| Help desk (password resets) | 40% of tickets | < 5% (passwordless) | 35% reduction |

---

## Những sai lầm phổ biến

### Top 10 Pitfalls khi triển khai Zero Trust

| # | Sai lầm | Hậu quả | Cách tránh |
|---|---------|---------|------------|
| 1 | **Big-bang approach** | Disruption lớn, resistance, failure | Phased approach, bắt đầu nhỏ |
| 2 | **Coi Zero Trust là sản phẩm** | Mua tool nhưng không thay đổi process | Zero Trust là strategy, không phải product |
| 3 | **Bỏ qua user experience** | Shadow IT tăng, workarounds | UX phải tốt hơn hoặc ngang trước |
| 4 | **Không có executive sponsorship** | Budget bị cắt, project stall | Secure C-suite buy-in trước |
| 5 | **Ignore legacy systems** | Gaps trong coverage | Plan cho legacy: wrap, migrate, or isolate |
| 6 | **Over-engineering từ đầu** | Complexity chưa cần thiết | Start simple, iterate |
| 7 | **Không đo lường** | Không chứng minh được value | Define KPIs từ Phase 1 |
| 8 | **Quên Non-Human Identities** | NHI là attack vector lớn nhất | NHI inventory + lifecycle management |
| 9 | **Treat network seg as sufficient** | Network ≠ Zero Trust | Identity-first, network là một layer |
| 10 | **Không plan cho AI workloads** | AI gap growing daily | Include AI security từ Phase 2+ |

---

## Organizational Change Management

### Zero Trust Impact Matrix

| Stakeholder | Impact | Mitigation |
|-------------|--------|-----------|
| **End Users** | New auth flows, device compliance | Training, better UX, self-service |
| **IT Ops** | New tools, processes, on-call changes | Phased training, automation |
| **Developers** | Service mesh, identity integration | DevSecOps enablement, templates |
| **Security Team** | Expanded scope, new capabilities | Upskilling, hiring, SOAR automation |
| **Leadership** | Budget, organizational changes | Regular reporting, business metrics |
| **Partners/Vendors** | New access requirements | Clear communication, transition support |

### Communication Plan

| Phase | Message | Audience | Channel |
|-------|---------|----------|---------|
| Pre-launch | "Why we're doing this" + timeline | All employees | Town hall, email |
| Phase 1 | Quick wins results, what's next | All employees | Newsletter |
| Phase 2 | User-facing changes (MFA, device) | End users | Training sessions |
| Phase 3 | Advanced capabilities, security posture improvement | Leadership | Executive briefing |
| Ongoing | Metrics, incidents prevented, ROI | Board | Quarterly report |

---

## Technology Stack Reference

### Vendor-Neutral Architecture Layers

| Layer | Function | Open Source Options | Commercial Options |
|-------|----------|--------------------|--------------------|
| **Identity Provider** | Authentication, SSO | Keycloak, Authentik | Entra ID, Okta, Ping |
| **Policy Engine** | Access decisions | OPA, Cedar | Styra, Axiomatics |
| **Service Mesh** | mTLS, traffic policy | Istio, Linkerd, Cilium | Consul Connect, AWS App Mesh |
| **Network Security** | Segmentation, filtering | Cilium, Calico | Illumio, Zscaler, Palo Alto |
| **ZTNA** | Application access | Headscale (Tailscale) | Zscaler ZPA, Cloudflare Access |
| **Secrets Management** | Credential lifecycle | HashiCorp Vault (BSL) | CyberArk, AWS Secrets Manager |
| **SIEM/XDR** | Detection, response | Wazuh, Elastic Security | Splunk, Microsoft Sentinel, CrowdStrike |
| **Data Protection** | Classification, DLP | OpenDLP | Microsoft Purview, Symantec |
| **Container Security** | K8s hardening | Falco, Trivy, Kyverno | Aqua, Prisma Cloud, Sysdig |
| **AI Security** | Agent monitoring, guardrails | Guardrails AI, LangKit | Microsoft ZT4AI, Cisco AI Defense |

---

## Tổng kết Series

### Bản đồ kiến thức Zero Trust 2026

```
Part 1: WHY Zero Trust?
├── Bối cảnh 2026, thống kê, xu hướng
└── Nguyên tắc nền tảng

Part 2: WHAT is Zero Trust Architecture?
├── NIST SP 800-207 — 7 tenets
├── PDP/PEP model
└── Deployment approaches

Part 3: WHO & WHERE — Identity + Segmentation
├── Identity-Centric Security (6 loại identity)
├── Micro-segmentation (workload-level)
└── Implementation patterns

Part 4: HOW to deliver — ZTNA/SASE/SSE
├── Network access evolution
├── SASE architecture
└── Vendor landscape

Part 5: Cloud-Native & Kubernetes
├── K8s-specific Zero Trust
├── Service mesh, RBAC, NetworkPolicy
└── Supply chain security

Part 6: AI Workloads (2026-specific)
├── Instruction-data collapse
├── Non-Human Identities at scale
├── Agent identity & behavioral monitoring
└── ZT4AI frameworks

Part 7: HOW to implement (bài này)
├── Assessment & maturity model
├── 4-phase roadmap
├── Cost/ROI
└── Change management
```

### Key Takeaways

1. **Zero Trust là hành trình, không phải đích đến.** Không có "done" — chỉ có increasingly mature.

2. **Bắt đầu với Identity.** 80% breach bắt đầu từ compromised credentials. Identity-first là non-negotiable.

3. **Phased approach wins.** Big-bang fails. Quick wins → prove value → expand scope.

4. **AI workloads cần attention ngay.** Nếu bạn đang deploy AI/LLMs, traditional Zero Trust là necessary nhưng NOT sufficient.

5. **Measure everything.** Không đo được = không chứng minh được = không có budget.

6. **UX matters.** Security mà users bypass = no security.

7. **Non-Human Identities là blind spot lớn nhất.** NHI outnumber humans 45:1 đến 500:1. Quản lý chúng.

---

## Nguồn tham khảo

- [CISA Zero Trust Maturity Model v2.0](https://www.cisa.gov/zero-trust-maturity-model)
- [NIST SP 800-207: Zero Trust Architecture](https://csrc.nist.gov/publications/detail/sp/800-207/final)
- [NIST SP 1800-35: Implementing a Zero Trust Architecture](https://www.nccoe.nist.gov/zerotrust)
- [Gartner 2025 Strategic Roadmap for Zero Trust Program Implementation](https://graygroupintl.com/blog/zero-trust-security-architecture)
- [Zero Trust Security in 2026: Architecture, Implementation Steps - Global Tech Council](https://www.globaltechcouncil.org/cyber-security/zero-trust-security-2026-architecture-implementation-steps-common-pitfalls/)
- [Forrester: The Zero Trust Implementation Guide](https://www.forrester.com/report/the-zero-trust-implementation-guide)
- [Microsoft Zero Trust Deployment Guide](https://learn.microsoft.com/en-us/security/zero-trust/deploy/overview)
- [DoD Zero Trust Reference Architecture v2](https://dodcio.defense.gov/Portals/0/Documents/Library/ZTRAv2.0.pdf)
- [IBM Cost of a Data Breach Report 2025](https://www.ibm.com/reports/data-breach)
- [Zero Trust Architecture: Enterprise Guide 2026 - Trantorinc](https://www.trantorinc.com/blog/zero-trust-architecture-guide-for-enterprises)
