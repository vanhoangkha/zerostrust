# Part 4: ZTNA, SASE và SSE — Network Access hiện đại

## Giới thiệu

Nếu Identity là "control plane" và Micro-segmentation là "data plane" của Zero Trust, thì **ZTNA, SASE, và SSE** là **delivery model** — cách Zero Trust được triển khai và vận hành trên quy mô enterprise.

Năm 2026, ba acronym này thống trị thị trường security networking. Nhưng ranh giới giữa chúng mờ nhạt, gây nhầm lẫn lớn. Bài viết này giải mã cách chúng thực sự hoạt động và relate với nhau.

## Giải mã thuật ngữ

### Bức tranh tổng thể

```
┌─────────────────────────────────────────────────────────────┐
│                          SASE                                │
│  (Secure Access Service Edge)                               │
│                                                             │
│  ┌────────────────────────┐  ┌────────────────────────────┐ │
│  │       SD-WAN           │  │         SSE                │ │
│  │  (Networking Layer)    │  │  (Security Service Edge)   │ │
│  │                        │  │                            │ │
│  │  - WAN Optimization    │  │  ┌──────┐ ┌─────┐ ┌────┐  │ │
│  │  - Traffic routing     │  │  │ ZTNA │ │ SWG │ │CASB│  │ │
│  │  - QoS                 │  │  └──────┘ └─────┘ └────┘  │ │
│  │  - Path selection      │  │  ┌─────┐  ┌─────┐ ┌────┐  │ │
│  │                        │  │  │ DLP │  │FWaaS│ │ RBI│  │ │
│  │                        │  │  └─────┘  └─────┘ └────┘  │ │
│  └────────────────────────┘  └────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Công thức đơn giản:**
- **SASE = SD-WAN + SSE** (Networking + Security)
- **SSE = ZTNA + SWG + CASB + DLP + FWaaS + RBI** (Security only)
- **ZTNA** là thành phần thay thế VPN trong SSE

## ZTNA — Zero Trust Network Access

### VPN vs ZTNA: Sự khác biệt cốt lõi

| Đặc điểm | Legacy VPN | ZTNA |
|-----------|-----------|------|
| Access model | Network-level (toàn mạng) | Per-application (từng app) |
| Authentication | One-time login | Continuous verification |
| Trust model | Trust after connect | Never trust, always verify |
| Visibility | User thấy toàn bộ network | Resources invisible until authorized |
| Lateral movement | Dễ dàng sau khi connected | Blocked — chỉ reach apps được policy cho phép |
| Attack surface | Inbound ports open | No inbound ports (service-initiated) |
| Scalability | Appliance-bound | Cloud-native, infinite scale |
| User experience | Full tunnel, slow | Split tunnel, per-app, fast |

### Cách ZTNA hoạt động

```
┌──────────┐          ┌─────────────────┐         ┌────────────┐
│  User    │ ──auth──▶│  ZTNA Broker    │◀──conn── │ Connector  │
│  Device  │          │  (Cloud/Edge)   │         │ (Near App) │
│          │          │                 │         │            │
│  Agent   │◀─tunnel─▶│  • Verify ID    │──proxy──▶│  Private   │
│  or      │          │  • Check device │         │  App       │
│  Browser │          │  • Apply policy │         │            │
└──────────┘          │  • Per-session  │         └────────────┘
                      └─────────────────┘
```

**Hai kiến trúc ZTNA:**

1. **Service-Initiated (Connector-based):**
   - Connector chạy near the app, kết nối outbound đến broker
   - Không cần mở inbound ports → app invisible cho scanners
   - Phổ biến nhất: Zscaler Private Access, Cloudflare Access

2. **Endpoint-Initiated (Agent-based):**
   - Agent trên user device mở authenticated tunnel đến broker
   - Cần agent installation nhưng hỗ trợ more protocols
   - Phổ biến: Tailscale, Twingate, Appgate

### ZTNA trong thực tế 2026

Theo Gartner, **60% giao dịch SD-WAN mới** tích hợp SASE single-vendor — đồng nghĩa ZTNA thay thế VPN đang tăng tốc mạnh.

**Use cases phổ biến:**
- Replace VPN cho remote access
- Secure access cho contractors/third-parties (không cần full VPN)
- Protect admin consoles và internal dashboards
- M&A integration — cấp access nhanh mà không cần merge networks

### Vendors tiêu biểu

| Vendor | Sản phẩm | Đặc điểm |
|--------|----------|----------|
| Zscaler | Private Access | Leader, service-initiated, largest PoP network |
| Cloudflare | Access/Tunnel | Developer-friendly, service-initiated, free tier |
| Palo Alto | Prisma Access | Enterprise, integrated SASE |
| Netskope | Private Access | Strong CASB integration |
| Tailscale | Tailscale | WireGuard-based, peer-to-peer, developer-loved |
| Twingate | Twingate | SDP model, easy deployment |
| Cisco | Duo + Secure Connect | Enterprise existing stack |

## SSE — Security Service Edge

### Tại sao SSE tách khỏi SASE?

Gartner tách SSE khỏi SASE năm 2021 vì nhận ra:
- **Network team ≠ Security team** — khác tổ chức, khác timeline procurement
- Security team cần move first mà **không cần replace toàn bộ WAN**
- SSE cho phép deploy security services trước, layer SD-WAN sau (hoặc không bao giờ)

### Các thành phần SSE

#### 1. ZTNA (đã mô tả ở trên)
Per-application access thay thế VPN.

#### 2. SWG — Secure Web Gateway

**Chức năng:** Proxy kiểm tra và filter outbound web traffic
- URL categorization
- Malicious destination blocking
- TLS decryption & inspection
- File scanning for malware
- Acceptable-use policy enforcement

**Tiến hóa:** Từ on-prem proxy (Blue Coat/Websense) → cloud-delivered service trong SSE.

#### 3. CASB — Cloud Access Security Broker

**Chức năng:** Control point cho SaaS applications (sanctioned và unsanctioned)
- Shadow IT discovery
- Policy enforcement cho M365, Salesforce, Workday
- Data exposure scanning (public S3 buckets, public Drive folders)
- DLP cho data flowing to/from cloud

**Hai modes:**
- **API-based:** CASB kết nối SaaS app's API → scan và enforce posture
- **Inline:** CASB trong traffic path → intercept real-time

#### 4. DLP — Data Loss Prevention

**Chức năng:** Kiểm soát outbound data across tất cả paths
- Content inspection (PII, credentials, sensitive data)
- Policy enforcement per channel (email, web, SaaS, endpoint)
- Integration với SWG + CASB

#### 5. FWaaS — Firewall as a Service

**Chức năng:** Cloud-delivered firewall cho non-web traffic
- Replace legacy branch firewalls
- L3/L4 filtering từ cloud
- Thường combined với SWG cho full traffic coverage

#### 6. RBI — Remote Browser Isolation

**Chức năng:** Browser chạy trong cloud sandbox, stream visual result về user
- Loại bỏ toàn bộ class browser-based attacks
- Drive-by downloads, malicious JS, browser zero-days → eliminated
- User thấy normal browsing experience

**Deployment models:**
- **Selective RBI:** Chỉ isolate risky/unknown sites (driven by SWG)
- **Full RBI:** Isolate all web traffic cho high-risk users (VIPs, finance)

### Vendors SSE tiêu biểu

| Vendor | Platform | Ghi chú |
|--------|----------|---------|
| Zscaler | Zero Trust Exchange | Market leader |
| Netskope | ONE Platform | Strong CASB/DLP |
| Palo Alto | Prisma Access | Integrated platform |
| Cloudflare | Cloudflare One | Developer-friendly, aggressive pricing |
| Cato Networks | Cato SASE Cloud | Single-pass architecture |

## SASE — Secure Access Service Edge

### SASE = SD-WAN + SSE

SASE converge **toàn bộ networking và security** vào một cloud-edge fabric:

```
┌──────────────────────────────────────────────────────────┐
│                    SASE Cloud Edge                        │
│                (Global Points of Presence)                │
├──────────────────────────────────────────────────────────┤
│                                                          │
│    User ──────┐                                          │
│               │                                          │
│    Branch ────┼──▶ [PoP] ──▶ Inspect ──▶ Route ──▶ App  │
│               │     │         │                          │
│    IoT ───────┘     │         ├─ Identity check          │
│                     │         ├─ Device posture          │
│                     │         ├─ DLP scan                │
│                     │         ├─ Threat detection        │
│                     │         └─ Policy enforcement      │
│                     │                                    │
│               SD-WAN Layer        SSE Layer              │
│               (Routing/QoS)       (Security)             │
└──────────────────────────────────────────────────────────┘
```

### Khi nào cần full SASE vs chỉ SSE?

| Tình huống | Lựa chọn |
|-----------|----------|
| Có nhiều branch offices cần WAN modernization | Full SASE |
| Đang dùng MPLS, muốn migrate dần | Full SASE |
| Chủ yếu remote users + cloud apps | SSE đủ |
| Đã có SD-WAN từ vendor khác | SSE + keep existing SD-WAN |
| Budget hạn chế, cần security first | SSE trước, SASE sau |

### Single-Vendor vs Best-of-Breed

**Single-Vendor SASE (xu hướng 2026):**
- Một vendor cho tất cả (Zscaler, Palo Alto, Cato)
- Ưu: Integrated experience, single pane of glass, simpler ops
- Nhược: Vendor lock-in, may not be best-in-class ở mọi category

**Best-of-Breed:**
- Mix vendors per capability (Okta + Cloudflare Access + Netskope CASB)
- Ưu: Best-in-class mỗi category
- Nhược: Integration overhead, multiple consoles, policy consistency challenges

## Deployment Strategy — Thứ tự triển khai thực tế

### Stage 0: Pre-Zero Trust
- Castle-and-moat: VPN, on-prem firewall, perimeter MFA
- Starting point cho hầu hết tổ chức

### Stage 1: Identity Foundation (0-6 tháng)
- Single IdP (Okta, Entra ID, Ping) cho tất cả apps
- MFA on every login, phishing-resistant cho high-value
- Device trust signals fed into access decisions
- **→ Delivers largest material risk reduction**

### Stage 2: ZTNA Replacing VPN (6-12 tháng)
- Chọn ZTNA platform
- Migrate sensitive apps trước: admin consoles, internal dashboards, dev environments
- Phase out VPN by department/app category — **không big-bang**
- Surfaces policy work: app inventory, who needs what, access rules

### Stage 3: SSE Foundation (12-18 tháng)
- Add cloud-delivered SWG cho outbound web filtering
- Add CASB cho SaaS visibility và policy
- Layer DLP across SWG + CASB + email
- **→ Security team có coherent cloud edge cho web + SaaS + private apps**

### Stage 4: Selective Enhancements (18+ tháng)
- RBI cho high-risk users/unknown sites
- FWaaS retire legacy branch firewalls
- SD-WAN → complete SASE (nếu WAN modernization cần)
- DEM (Digital Experience Monitoring) cho user-experience

### Stage 5: Continuous Improvement
- Policy refinement from usage telemetry
- Microsegmentation inside datacenter
- PAM integrated vào ZT policy framework
- Attack surface monitoring

## Xu hướng 2026-2027

### 1. ZTNA + Identity đang converge

ZTNA platforms ngày càng **indistinguishable from IdP-extended policy enforcement**:
- Cloudflare, Tailscale, Okta → pushing IdP-centric enforcement model
- Endgame: ZTNA, SWG, CASB đều là features của **identity fabric** thay vì separate platforms

### 2. Enterprise Browser — Endpoint mới

Island và Talon (Palo Alto) tiên phong **enterprise browser** as primary access surface:
- Combine ZTNA + DLP + Browser Isolation + Policy → trong managed browser
- Compress nhiều SSE controls vào một client
- 2026: Category đang grow nhanh

### 3. AI-Driven Policy

Manual policy authoring = operational ceiling cho Zero Trust adoption:
- AI-assisted policy recommendations: analyze actual access patterns → suggest tighter policies
- Zscaler, Cloudflare, Netskope đều đang ship AI policy features
- Giảm Policy Paralysis — failure pattern phổ biến nhất

## Ba patterns dự đoán thành công

### 1. Identity treated as critical infrastructure
- Dedicated identity team
- Single primary IdP
- MFA everywhere, **no exceptions for senior leadership**

### 2. Policies owned by application owners
- Centralized security team **cannot scale** policy maintenance across thousands apps
- Delegate policy authoring → app owners
- Security team sets guardrails, reviews exceptions
- ZTNA platforms with policy-as-code support (Cloudflare, Tailscale) enable this

### 3. Measured in outcomes, not products purchased
```
❌ "We deployed SASE"                    (procurement milestone)
✓ "We retired the legacy VPN"           (outcome)
✓ "MFA coverage is 98% of identity events" (outcome)
✓ "Mean time to provision access: 4 hours" (outcome)
```

## Zero Trust ≠ SASE

Cần phân biệt rõ:
- **SASE là delivery model** cho Zero Trust enforcement
- Zero Trust **không** yêu cầu SASE
- SASE **không tự động** = Zero Trust
- Organizations có thể làm ZT without SASE (on-prem heavy, data-locality requirements)
- Organizations có thể mua SASE without coherent identity/policy → **không phải ZT**

## Tổng kết

| Concept | Vai trò | Analogy |
|---------|---------|---------|
| Zero Trust | Nguyên tắc/Strategy | "Luật giao thông" |
| NIST 800-207 | Kiến trúc chuẩn | "Bản vẽ thiết kế" |
| SASE | Delivery platform (Network + Security) | "Con đường + camera + rào chắn" |
| SSE | Security services only | "Camera + rào chắn" (không có đường) |
| ZTNA | VPN replacement | "Cổng vào có kiểm soát danh tính" |
| SWG/CASB/DLP | Data protection controls | "Bộ lọc tại mỗi ngã rẽ" |

**Thứ tự quan trọng:** Identity first → ZTNA → SSE → SASE (nếu cần). Đừng nhảy đến Stage 4 vì vendor demo impressive. Foundation identity và ZTNA là thứ **làm mọi thứ khác useful**.

Trong phần tiếp theo, chúng ta sẽ đi sâu vào **Zero Trust cho Cloud Native & Kubernetes** — nơi 89% tổ chức đã gặp security incidents trong năm qua.

---

## Nguồn tham khảo

- [Zero Trust Architecture Explained: SASE, SSE, ZTNA - Deepak Gupta](https://guptadeepak.com/research/zero-trust-architecture-explained/)
- [ZTNA Complete Guide 2026 - Calmops](https://calmops.com/network/zero-trust-network-access-ztna-complete-guide/)
- [Zero Trust vs SASE Enterprise Network Security 2026 - Ciphers Security](https://cipherssecurity.com/zero-trust-vs-sase-enterprise-network-security/)
- [SASE vs ZTNA - Check Point](https://www.checkpoint.com/cyber-hub/network-security/what-is-secure-access-service-edge-sase/sase-vs-ztna/)
- [What is ZTNA? - Netskope](https://www.netskope.com/security-defined/what-is-zero-trust-network-access)
- [What is ZTNA? - Zscaler](https://www.zscaler.com/learn/zero-trust-network-access-ztna)
- [Zero Trust Security 2026: CIO Implementation - Transputec](https://www.transputec.com/blogs/zero-trust-security-2026-cio-implementation/)
