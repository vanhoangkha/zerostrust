# Part 3: Identity-Centric Security & Micro-segmentation

## Giới thiệu

Nếu NIST SP 800-207 là lý thuyết, thì **Identity-Centric Security** và **Micro-segmentation** là hai trụ cột thực hành quan trọng nhất khi triển khai Zero Trust. Chúng trả lời hai câu hỏi cốt lõi:

- **Identity:** "AI/Cái gì đang yêu cầu truy cập?" → Xác minh danh tính
- **Micro-segmentation:** "Nếu bị xâm nhập, thiệt hại lan rộng đến đâu?" → Giới hạn blast radius

Năm 2026, identity đã trở thành **primary control plane** cho Zero Trust, và micro-segmentation đã vượt xa network segments truyền thống để đạt đến **workload-level enforcement**.

## Phần 1: Identity-Centric Security

### Identity là Control Plane mới

Trong Zero Trust 2026, identity không chỉ là username/password. Nó bao gồm:

| Loại Identity | Ví dụ | Đặc điểm 2026 |
|---|---|---|
| Workforce Identity | Nhân viên, contractors | Phishing-resistant MFA, passwordless |
| Service Account | Backend services | Short-lived tokens, auto-rotation |
| API Identity | External/internal APIs | OAuth 2.0 + mTLS |
| Container Identity | Pods, workloads | SPIFFE/SPIRE certificates |
| Machine Identity | IoT, devices | Platform attestation |
| AI Agent Identity | LLM agents, copilots | Scoped permissions, audit trail |

### Nguyên tắc Identity-Centric Security

#### 1. Strong Authentication (Xác thực mạnh)

```
Mức độ yếu → mạnh:

Password only          → ❌ Không chấp nhận
Password + SMS OTP     → ⚠️  Phishing-susceptible
Password + TOTP app    → ✓  Acceptable baseline
Passkey (FIDO2/WebAuthn) → ✓✓ Phishing-resistant
Hardware key (YubiKey)    → ✓✓✓ Strongest
```

**Năm 2026:** Phishing-resistant MFA (FIDO2/WebAuthn/Passkeys) là minimum requirement cho privileged access.

#### 2. Continuous Identity Verification

Không chỉ verify lúc login — verify liên tục dựa trên:
- **Behavioral analytics:** Pattern truy cập bất thường
- **Device posture changes:** Device bị jailbreak, thiếu patch
- **Location anomalies:** Impossible travel detection
- **Session risk scoring:** Confidence score thay đổi → re-authenticate hoặc terminate

#### 3. Least Privilege & Just-In-Time Access

```
Truyền thống:
  Admin account → permanent admin rights → 24/7 access

Zero Trust 2026:
  User → request elevated access → approval workflow
       → time-bound privilege (2 hours) → auto-revoke
       → full audit trail
```

**Just-In-Time (JIT) Access:**
- Quyền elevated chỉ cấp khi cần
- Auto-expire sau thời gian định trước
- Approval workflow tích hợp

**Just-Enough Access (JEA):**
- Không cấp full admin
- Chỉ cấp chính xác permissions cần thiết cho task cụ thể

#### 4. Identity Governance

- **Joiner-Mover-Leaver (JML):** Tự động provision/de-provision access khi nhân viên join/move/leave
- **Access reviews:** Định kỳ review và thu hồi access không cần thiết
- **Privilege creep detection:** Phát hiện accumulation of access over time
- **Orphaned accounts:** Tự động detect và disable

### Workload & Machine Identity

Đây là **gap lớn nhất** mà nhiều tổ chức bỏ qua. Focus chỉ user MFA nhưng bỏ quên service-to-service → lateral movement vẫn dễ dàng.

#### SPIFFE/SPIRE — Tiêu chuẩn Workload Identity

**SPIFFE** (Secure Production Identity Framework for Everyone):
- Cung cấp cryptographic identity cho mọi workload
- Identity dạng URI: `spiffe://trust-domain/workload-identifier`
- Không phụ thuộc vào network location hay IP

**SPIRE** (SPIFFE Runtime Environment):
- Implementation của SPIFFE
- Auto-issue và rotate short-lived X.509 certificates
- Platform attestation (verify workload đang chạy trên đúng platform)

#### Short-Lived Credentials — Nguyên tắc bắt buộc

```
❌ Static secrets:
   API_KEY=sk-abc123...  (valid forever, stored in .env)
   
✓ Short-lived credentials:
   Token issued at 14:00, expires at 14:15
   Auto-rotated, never stored permanently
   Tied to specific workload identity
```

**Tại sao short-lived credentials quan trọng:**
- Static secrets bị leak → attacker có access vĩnh viễn
- Short-lived credentials bị leak → window of exposure = minutes, không phải months

### Identity-Centric Architecture Pattern

```
┌─────────────────────────────────────────────────────┐
│                 IDENTITY PROVIDER (IdP)              │
│  ┌───────────┐  ┌──────────┐  ┌─────────────────┐  │
│  │ User Auth │  │ Machine  │  │ Policy Engine   │  │
│  │ (FIDO2)   │  │ Identity │  │ (Adaptive)      │  │
│  │           │  │ (SPIFFE) │  │                 │  │
│  └─────┬─────┘  └─────┬────┘  └────────┬────────┘  │
└────────┼───────────────┼────────────────┼───────────┘
         │               │                │
    ┌────▼────┐    ┌─────▼─────┐    ┌─────▼─────┐
    │  Token  │    │Certificate│    │  Access   │
    │ (JWT)   │    │ (X.509)   │    │ Decision  │
    └────┬────┘    └─────┬─────┘    └─────┬─────┘
         │               │                │
    ┌────▼───────────────▼────────────────▼────┐
    │         POLICY ENFORCEMENT POINTS         │
    │  API Gateway | Service Mesh | App Layer   │
    └──────────────────────────────────────────┘
```

## Phần 2: Micro-segmentation

### Vấn đề: East-West Traffic không được kiểm soát

Perimeter firewalls bảo vệ **north-south traffic** (internet ↔ data center). Nhưng một khi attacker ở bên trong (phished credentials, compromised vendor, hijacked CI runner):

- Mạng nội bộ thường **flat** — workloads giao tiếp tự do
- **Implicit trust** cho mọi thứ inside perimeter
- Một laptop compromise → ransomware trên file servers
- Một credential leak → tuần/tháng intrusion không phát hiện

**Micro-segmentation loại bỏ implicit trust** bằng cách yêu cầu mỗi east-west connection được **explicitly authorized by policy**, không phải by network topology.

### Ba cấp độ Segmentation

| Cấp độ | Mô tả | Ví dụ | Zero Trust Level |
|---|---|---|---|
| **Macro-segmentation** | Chia thành broad zones | DMZ, internal, dev, prod, PCI scope | Necessary but not sufficient |
| **Micro-segmentation** | Per-workload/per-service zones | Web tier → chỉ nói chuyện với DB tier | Strong foundation |
| **Application-aware** | Policy theo authenticated identity + API path | Service A chỉ GET /api/orders, không DELETE | Optimal (CISA ZTMM) |

### Implementation Paths

#### 1. SDN Overlays (VMware NSX, Cisco ACI)

- Policy trong network controller, enforcement tại hypervisor
- **Phù hợp:** Data centers trưởng thành, VMware estates
- **Hạn chế:** Khó áp dụng cho cloud-native (hypervisor bị ẩn)

#### 2. Host-Agent Micro-segmentation (Illumio, Akamai Guardicore)

- Agent trên mỗi workload, enforcement independent of network topology
- **Phù hợp:** Hybrid estates, Windows-heavy environments
- **Ưu điểm:** Observe-only mode trước khi enforce; roll out incrementally

#### 3. Cloud-Native Security Groups (AWS SG, Azure NSG, GCP Firewall)

- Platform đã có sẵn, chi phí thấp nhất
- **Phù hợp:** 80% segmentation work trong cloud
- **Hạn chế:** Thiếu identity-aware policy, khó cross-cloud consistency

#### 4. Service Mesh (Istio, Linkerd, Consul Connect)

- Identity-aware segmentation tại service-to-service layer
- mTLS giữa mọi service, policy theo service identity
- **Phù hợp:** Cloud-native/Kubernetes workloads
- **Hạn chế:** Không relevant cho file servers, ERP systems

### Kết hợp thực tế

Hầu hết tổ chức regulated kết hợp **2-3 approaches**:

```
┌─────────────────────────────────────────────────┐
│          ENTERPRISE SEGMENTATION STACK          │
├─────────────────────────────────────────────────┤
│                                                 │
│  Cloud-native apps:  Service Mesh (Istio)       │
│                      + K8s NetworkPolicy         │
│                                                 │
│  Cloud workloads:    Security Groups/NSGs       │
│                      + Host-agent (Illumio)      │
│                                                 │
│  On-prem/legacy:     SDN Overlay (NSX)          │
│                      + Host-agent               │
│                                                 │
│  Data center:        Macro-segmentation (VLANs) │
│                      + Micro-seg (Guardicore)    │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Rollout Sequence — Cách triển khai thực tế

#### Bước 1: Map actual traffic (30+ ngày)

**Không bao giờ bắt đầu bằng viết policy.** Bắt đầu bằng visibility.

- Deploy flow-log/agent ở observe-only mode
- Thu thập actual east-west traffic patterns tối thiểu 30 ngày
- Policy viết ngày đầu tiên **sẽ sai** — policy từ 30 ngày real traffic sẽ defensible

#### Bước 2: Group workloads theo application + data sensitivity

Đơn vị phòng thủ = **application boundary**:
- Nhóm services này
- Nói chuyện trên ports này
- Với database này
- Cho user populations này

#### Bước 3: Viết policy ở observe-only mode

Mọi tool segmentation hiện đại đều hỗ trợ **non-blocking phase**:
- Policy được log nhưng không enforce
- Phát hiện các calls bị bỏ sót trong flow analysis

#### Bước 4: Enforce theo từng cohort

- Switch một application sang enforce mode
- Watch incidents 2 tuần
- Di chuyển sang application tiếp theo
- **Big-bang enforcement = lý do #1 segmentation programs stall**

#### Bước 5: Chuyển sang identity-based rules

```
❌ IP-based:  "10.0.4.0/24 can reach 10.0.6.0/24"
   → Breaks khi workload di chuyển

✓ Identity-based: "order-service can read orders-db"
   → Survives workload migration
```

#### Bước 6: Tích hợp policy vào CI/CD

- Policy-as-Code (OPA Rego, Terraform, Helm)
- Policy trong Git, reviewed qua PR process
- Auto-deploy cùng application code
- **Manual policy update = policy drift**

### Metrics đo hiệu quả

| Metric | Mục tiêu |
|--------|----------|
| East-west traffic crossing zone boundary | Giảm dần, ổn định tại known authorized paths |
| Policy denies/workload/tuần | Số nhỏ, giảm dần, có owner cho mỗi unexpected deny |
| Mean time to update policy cho new service | Giờ, không phải tuần |

### Thất bại phổ biến

| Pattern | Mô tả | Giải pháp |
|---------|--------|----------|
| Top-down policy không có traffic data | Viết policy từ network diagram → break apps khi enforce | Flow analysis 30 ngày trước |
| Network team owns it alone | Không có input từ application owners → brittle policy | Pair network + app teams |
| Exception accumulation | 1 emergency exception → default path | Time-bound exceptions, monthly audit |
| IP rules survive identity mandate | Database firewall vẫn hardcoded subnet | Migrate to identity-based rules |
| No telemetry on denies | Không biết policy quá tight hay quá loose | Pipe logs vào SIEM |

## Phần 3: Identity + Micro-segmentation = Zero Trust Control Plane

### Kết hợp hai pillars

Identity-Centric Security và Micro-segmentation không hoạt động riêng lẻ — chúng **kết hợp** để tạo Zero Trust control plane hoàn chỉnh:

```
Request Flow:
                                                        
  User/Service ──→ Authenticate (Identity) ──→ Authorize (Policy)
       │                                            │
       │                                            ▼
       │                                    Check: Is this identity
       │                                    allowed to cross THIS
       │                                    segment boundary to
       │                                    access THIS resource?
       │                                            │
       │                                            ▼
       └──────────────── PEP Enforcement ────────────┘
                         (Allow/Deny)
```

### Service Mesh — Điểm hội tụ

Service mesh (Istio, Linkerd) là nơi identity và segmentation **gặp nhau**:

1. **Automated mTLS:** Tự động provision và rotate certificates cho mọi service
2. **L7 Policy Enforcement:** "Service A chỉ gọi Service B's GET method" — granular hơn IP rules
3. **Service Identity:** Mỗi service có cryptographic identity (SPIFFE)
4. **Observability:** Unified source of truth cho all traffic

### The DevSecOps Mandate

ZTA trong microservices **không phải** security project — là **DevSecOps mandate**:

1. **Automated Identity Provisioning:** Service mesh/cloud-native IdP tự động issue và rotate service identities
2. **Policy-as-Code (PaC):** Declarative policies (YAML, OPA Rego) trong Git, deploy qua CI/CD
3. **Shift-Left Security:** Policy validation trong build/test phase, không đợi production
4. **Centralized Observability:** Aggregate security logs → continuous verification → auto-alerts

### Zero Trust Maturity Model cho Identity + Segmentation

| Stage | Focus | Engineering Goal |
|-------|-------|-----------------|
| 1. Initial | Inventory & Visibility | Document services, APIs, flows. Basic mTLS |
| 2. Segmented | Network Micro-segmentation | K8s NetworkPolicy, isolate major groups |
| 3. Identity-Aware | Service Mesh & mTLS | Service identity, granular L7 policies |
| 4. Automated | Policy-as-Code | OPA, dynamic access, secrets management |
| 5. Optimized | Continuous Verification | AI/ML anomaly detection, near-zero implicit trust |

## Gartner & Industry Validation 2026

- **Gartner 2026:** Microsegmentation là **foundational Zero Trust control** — core way to address security gap tại workload level
- **GigaOm Radar 2026:** Đánh giá 15 leading vendors trong microsegmentation
- **Illumio:** Dẫn đầu competitive landscape cho Network Security Microsegmentation
- **SANS 2026:** White paper đánh giá micro-segmentation trong Windows Active Directory environments
- **IBM 2026:** Microsegmentation = enforcing least privilege at workload level, beyond traditional segmentation

## Tổng kết

### Key Takeaways

1. **Identity là control plane chính** — bao gồm users, services, APIs, containers, AI agents
2. **Phishing-resistant MFA** là minimum cho privileged access
3. **Workload identity** (SPIFFE/SPIRE, short-lived credentials) không phải optional
4. **Micro-segmentation bắt đầu từ visibility**, không phải policy
5. **30 ngày observe → enforce theo cohorts** — không big-bang
6. **Identity-based rules** thay thế IP-based rules
7. **Policy-as-Code** tích hợp CI/CD — manual = drift
8. **Service mesh** = điểm hội tụ identity + segmentation cho cloud-native

Trong phần tiếp theo, chúng ta sẽ khám phá **ZTNA, SASE và SSE** — cách Zero Trust được delivery cho network access hiện đại.

---

## Nguồn tham khảo

- [Zero Trust Architecture Implementation Guide for Microservices - Developers.dev](https://www.developers.dev/tech-talk/the-operational-playbook-for-zero-trust-architecture-in-microservices-a-strategy-and-execution-guide.html)
- [Micro-Segmentation in Zero Trust Architecture - Pilotcore](https://pilotcore.io/blog/micro-segmentation-in-zero-trust-architecture)
- [Microsegmentation Explained - IBM](https://www.ibm.com/think/insights/microsegmentation-lateral-movement)
- [Gartner Competitive Landscape for Microsegmentation 2026 - Illumio](https://www.illumio.com/blog/gartner-competitive-landscape-for-network-security-microsegmentation-2026)
- [2026 GigaOm Radar for Microsegmentation - Zero Networks](https://zeronetworks.com/resource-center/reports/2026-gigaom-radar-for-microsegmentation)
- [Implementing Micro-Segmentation in Legacy Enterprise Lab - SANS](https://www.sans.org/white-papers/implementing-micro-segmentation-legacy-enterprise-lab-network)
- [Zero Trust Security in 2026 - Global Tech Council](https://www.globaltechcouncil.org/cyber-security/zero-trust-security-2026-architecture-implementation-steps-common-pitfalls/)
- [CISA Zero Trust Maturity Model v2.0](https://www.cisa.gov/sites/default/files/2023-04/zero_trust_maturity_model_v2_508.pdf)
