# Part 2: Kiến trúc Zero Trust theo NIST SP 800-207

## Giới thiệu

NIST Special Publication 800-207 là **tài liệu tham chiếu chuẩn mực** (canonical reference) cho Zero Trust Architecture. Được xuất bản tháng 8/2020, đây là nền tảng mà mọi framework Zero Trust khác — từ CISA Maturity Model đến DoD Reference Architecture — đều dựa trên.

Hiểu rõ NIST SP 800-207 không phải là lựa chọn — đó là **yêu cầu bắt buộc** cho bất kỳ ai thiết kế hoặc triển khai kiến trúc Zero Trust nghiêm túc.

## Lịch sử phát triển Zero Trust

| Năm | Sự kiện |
|------|---------|
| 2007 | Jericho Forum công bố Commandments về "deperimeterization" |
| 2010 | John Kindervag (Forrester) đề xuất mô hình Zero Trust |
| 2018 | Zero Trust trở thành marketing label cho mọi sản phẩm security |
| 2020 | NIST xuất bản SP 800-207 — chính thức hóa kiến trúc |
| 2021 | CISA phát hành Zero Trust Maturity Model v1 |
| 2022 | DoD phát hành Zero Trust Reference Architecture + Strategy |
| 2023 | CISA cập nhật Maturity Model v2.0 |
| 2025 | NIST SP 1800-35: Implementing Zero Trust Architecture (final) |
| 2026 | ETSI TS 104 102 (ZT-Kipling methodology) được áp dụng rộng rãi |

## Bảy Nguyên lý (Tenets) của Zero Trust theo NIST

NIST SP 800-207 định nghĩa Zero Trust thông qua **7 nguyên lý nền tảng**. Đây không phải best practices hay recommendations — chúng là **định nghĩa chính thức** của mô hình. Bất kỳ kiến trúc nào không tuân thủ các nguyên lý này đều không thực sự là Zero Trust.

### Tenet 1: Mọi nguồn dữ liệu và dịch vụ đều là tài nguyên (Resource)

> Laptop, ứng dụng SaaS, API endpoint, IoT sensor — tất cả đều là resource chịu cùng quy tắc kiến trúc.

Không có sự phân biệt giữa "thiết bị quan trọng" và "thiết bị phụ". Mọi thứ đều cần bảo vệ.

### Tenet 2: Mọi giao tiếp phải được bảo mật, bất kể vị trí mạng

> Traffic nội bộ phải có cùng yêu cầu authentication và encryption như traffic qua public internet.

**Đây là nguyên lý loại bỏ hoàn toàn khái niệm "trusted network".**

### Tenet 3: Truy cập tài nguyên được cấp trên cơ sở từng session

> Authentication cho một session không cấp quyền truy cập resources khác hoặc sessions tương lai.

Mỗi request = một quyết định truy cập mới.

### Tenet 4: Truy cập được xác định bởi dynamic policy

> Policy xem xét: identity, device state, behavioral analytics, environmental conditions, và context khác — không chỉ credentials.

Policy không tĩnh — nó thay đổi dựa trên ngữ cảnh thời gian thực.

### Tenet 5: Doanh nghiệp giám sát và đo lường integrity/security posture của mọi asset

> Không thiết bị nào được "miễn trừ vĩnh viễn" dựa trên compliance trước đó.

Liên tục đánh giá, không bao giờ tin tưởng vĩnh viễn.

### Tenet 6: Authentication và authorization là dynamic, được thực thi nghiêm ngặt trước khi cho phép truy cập

> Quyết định được đánh giá lại khi context thay đổi trong suốt session.

Đây là "continuous verification" — không phải verify một lần rồi bỏ qua.

### Tenet 7: Thu thập tối đa thông tin về trạng thái hiện tại của assets, hạ tầng, và communications

> Sử dụng thông tin thu thập để cải thiện security posture.

Tenet 7 đóng vòng lặp feedback — dữ liệu thu thập → cải thiện policy → bảo mật tốt hơn.

## Kiến trúc Logic: PDP và PEP

### Mô hình tổng thể

```
┌─────────────────────────────────────────────────────────┐
│                  POLICY DECISION POINT (PDP)             │
│  ┌──────────────────┐    ┌────────────────────────┐     │
│  │   Policy Engine  │    │  Policy Administrator  │     │
│  │                  │    │                        │     │
│  │  - Identity      │    │  - Issue tokens        │     │
│  │  - Device posture│    │  - Configure sessions  │     │
│  │  - Behavior      │    │  - Translate decisions  │     │
│  │  - Threat intel  │    │                        │     │
│  │  - Context       │    │                        │     │
│  └──────────────────┘    └────────────────────────┘     │
└───────────────────────────────┬─────────────────────────┘
                                │ Allow/Deny
                                ▼
┌─────────────────────────────────────────────────────────┐
│              POLICY ENFORCEMENT POINT (PEP)              │
│                                                         │
│  Gateway | Proxy | Agent | Sidecar | API Gateway        │
│                                                         │
└───────────────┬─────────────────────────┬───────────────┘
                │                         │
        ┌───────▼───────┐         ┌───────▼───────┐
        │    Subject    │         │   Resource    │
        │ (User/Device/ │         │ (App/Data/    │
        │  Service)     │         │  API/Service) │
        └───────────────┘         └───────────────┘
```

### Policy Decision Point (PDP)

PDP là nơi **quyết định truy cập được đưa ra**. Gồm 2 thành phần:

**Policy Engine:**
- Đánh giá request dựa trên policy hiện hành
- Inputs: identity attributes, device posture, time, location, behavioral analytics, threat intelligence
- Output: Allow hoặc Deny

**Policy Administrator:**
- Dịch quyết định thành dạng enforcement components có thể áp dụng
- Ví dụ: phát hành token, cấu hình session, thiết lập tunnel

### Policy Enforcement Point (PEP)

PEP là thành phần **thực thi quyết định** — đứng giữa subject và resource:
- Chặn mọi request
- Hỏi PDP: "Cho phép hay từ chối?"
- Thực thi quyết định

**Các hình thức PEP trong thực tế:**
- Gateway/Reverse proxy
- Endpoint agent
- Service mesh sidecar (Envoy/Istio)
- API gateway
- Application-level middleware
- Network firewall với identity awareness

> **Nguyên tắc scale:** Một PDP có thể phục vụ hàng ngàn PEPs phân tán khắp enterprise.

## Trust Algorithm — Cách đưa ra quyết định

NIST không quy định thuật toán cụ thể, nhưng mô tả 2 cách tiếp cận:

### 1. Criteria-based (Dựa trên tiêu chí)

Kiểm tra xem các điều kiện cụ thể có được đáp ứng không:
- Certificate hợp lệ trên device
- Patch level hiện tại (qua MDM)
- Địa lý origin trong danh sách approved
- Thời gian truy cập trong window cho phép
- Role phù hợp với resource

**Ưu điểm:** Dễ implement, dễ audit. Compliance teams thích.

### 2. Score-based (Dựa trên điểm tin cậy)

Mỗi input đóng góp weighted value → tổng hợp thành confidence score → so sánh với threshold.

**Ví dụ:**
```
Strong MFA          : +30 points
Healthy device      : +25 points
Normal behavior     : +20 points
Known location      : +15 points
Role-consistent     : +10 points
─────────────────────────────
Total               : 100/100 → Access granted

Compromised device  : -25 points
Unknown location    : -15 points
─────────────────────────────
Total               : 60/100 → Below threshold → Denied
```

### Thực tế triển khai

Hầu hết enterprise **kết hợp cả hai**:
- **Criteria:** Hard requirements (VD: không có clearance → không truy cập classified data)
- **Score:** Contextual nuances (VD: "something feels off" signals từ behavioral analytics)

**Caching & Triggers:**
- Lý thuyết: re-evaluate liên tục
- Thực tế: cache decisions cho short windows, re-evaluate khi triggers fire:
  - Session timeout
  - Device posture change
  - Geographic anomaly
  - Request resource nhạy cảm hơn trước

## Implicit Trust Zone — Khái niệm quan trọng nhất

### Định nghĩa

**Implicit Trust Zone** = vùng trong kiến trúc nơi subjects được tin tưởng mặc định sau một sự kiện authentication ban đầu.

### Perimeter Security vs Zero Trust

| Perimeter Security | Zero Trust |
|---|---|
| Trust được cấp tại network entry | Trust được đánh giá per-request |
| Implicit trust zone = toàn bộ internal network | Implicit trust zone = single interaction |
| Lateral movement cho phép mặc định | Lateral movement cần fresh authorization |
| Static defenses tại edge | Dynamic policy decisions từ current context |

### Tại sao lateral movement là killer

Mọi breach lớn đều có cùng pattern:
1. **Initial compromise** (phishing, vendor bị xâm nhập, laptop bị đánh cắp)
2. **Lateral movement** không bị phát hiện (tuần/tháng)
3. **Data exfiltration** hoặc ransomware

Zero Trust không ngăn initial compromise (điều đó luôn xảy ra) — nhưng **nén lateral movement window** đủ nhỏ để phát hiện và ngăn chặn trước khi attacker đến được target data.

## Ba Variants của Zero Trust Architecture

NIST mô tả 3 biến thể — không variant nào "đúng hơn" variant khác. Chúng đại diện cho các cách tiếp cận khác nhau cùng một lý thuyết.

### 1. Identity-Based Zero Trust

- Xây dựng kiến trúc xoay quanh **user identity** là trust signal chính
- Strong authentication + continuous identity verification + behavioral analytics
- **Phù hợp:** Enterprise với identity programs trưởng thành, workforce phân tán

### 2. Microsegmentation-Based Zero Trust

- Chia mạng thành nhiều zones nhỏ, mỗi zone có PEP riêng
- Lateral movement giữa zones cần fresh authorization
- **Phù hợp:** ICS/OT environments, defense contractors, utilities

### 3. Software-Defined Perimeter (SDP)

- Ẩn resources sau authentication proxy
- Resources **invisible** cho scanners và unreachable cho attackers cho đến khi subject được authenticated
- **Phù hợp:** Cloud-first organizations, đây là cơ sở cho hầu hết sản phẩm ZTNA

### Thực tế

Hầu hết enterprise **kết hợp elements từ multiple variants**. Điều quan trọng là 7 tenets được tuân thủ và implicit trust zone thu nhỏ đến mức tối thiểu.

## Building Blocks của một chương trình Zero Trust trưởng thành (2026)

Theo Global Tech Council và các industry guides, một chương trình Zero Trust hiệu quả năm 2026 bao gồm:

### 1. Identity & Access Management (IAM)
- Central identity providers
- Strong authentication (phishing-resistant MFA)
- Adaptive access
- Governance để giảm privilege creep

### 2. Device Posture & Endpoint Controls
- Access decisions tích hợp device health, patch level, security agent status
- Block hoặc step-up authentication khi endpoint thiếu patches

### 3. Workload & Machine Identity
- Cryptographic identity cho services và workloads
- Platform attestation
- **Short-lived credentials** (không static secrets)

### 4. Microsegmentation
- Service-level segmentation cho east-west traffic
- Allow-list policies dựa trên verified identities

### 5. Application-Layer Access (ZTNA)
- Kết nối specific users → specific applications
- Thay thế broad network access (VPN)

### 6. Data-Centric Security
- Classification + encryption + tokenization
- Attribute-based access control (ABAC) tại data access points

### 7. Telemetry & Continuous Monitoring
- Centralized logging: access decisions, authentications, posture changes
- Analytics + behavior baselines → detect anomalies → tune policies

## Những điều Zero Trust KHÔNG nói

### "Loại bỏ firewall và VPN"

Không. Zero Trust thay đổi **vai trò kiến trúc** của chúng:
- Firewall vẫn hữu ích cho segmentation
- VPN vẫn encrypt traffic
- Nhưng chúng không còn là **trust boundary chính**

### "Rip and replace toàn bộ hạ tầng"

NIST và CISA đều nhận ra Zero Trust là **hành trình nhiều năm**. CISA Maturity Model v2.0 định nghĩa 4 giai đoạn:
1. **Traditional** — Perimeter-based, implicit trust
2. **Initial** — Bắt đầu áp dụng một số nguyên tắc ZT
3. **Advanced** — Tự động hóa, integration cross-pillars
4. **Optimal** — Fully dynamic, data-driven decisions

### "Không tin tưởng con người"

Zero Trust nhắm vào **implicit trust ở cấp kiến trúc**, không phải đánh giá về sự đáng tin cậy của user. Bất kỳ system/account/device nào cũng có thể bị compromise → không grant access dựa trên assumed trustworthiness.

## NIST SP 800-207 trong bối cảnh 2026

### Mở rộng cho AI Workloads

NIST SP 800-207 **ra đời trước LLMs, AI agents, và inference pipelines**. Năm 2026, nó được mở rộng với 6 trust boundaries mới cho AI:

1. User → Inference API
2. Inference → Model
3. Model → Tools
4. Tools → Data
5. Agent → Agent
6. Pipeline → External Services

### Frameworks liên quan

| Framework | Vai trò |
|-----------|---------|
| NIST SP 800-207 | Kiến trúc nền tảng (canonical reference) |
| NIST SP 1800-35 | Hướng dẫn triển khai thực tế |
| CISA Zero Trust Maturity Model v2.0 | Đánh giá mức độ trưởng thành |
| DoD ZT Reference Architecture | Ứng dụng cho quốc phòng |
| ETSI TS 104 102 (ZT-Kipling) | Phương pháp luận lặp (iterative) |
| Executive Order 14028 | Mandate FY2027 cho federal agencies |

## Tổng kết

> **Zero Trust là mô hình kiến trúc bảo mật được định nghĩa bởi NIST SP 800-207 qua 7 nguyên lý cụ thể. Nó giả định không có implicit trust nào được cấp dựa trên network location, identity alone, hay device ownership. Mọi quyết định truy cập được đánh giá dynamically bởi PDP và thực thi bởi PEP, với mục tiêu nén implicit trust zone xuống footprint nhỏ nhất có thể.**

Trong phần tiếp theo, chúng ta sẽ đi sâu vào **Identity-Centric Security và Micro-segmentation** — hai building blocks quan trọng nhất trong triển khai Zero Trust thực tế.

---

## Nguồn tham khảo

- [NIST SP 800-207 - Zero Trust Architecture](https://csrc.nist.gov/publications/detail/sp/800-207/final)
- [NIST SP 1800-35 - Implementing a Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/1800/35/final)
- [What NIST 800-207 Actually Defines - Training Camp](https://trainingcamp.com/articles/zero-trust-theory-what-nist-800-207-actually-defines/)
- [Zero Trust Security in 2026 - Global Tech Council](https://www.globaltechcouncil.org/cyber-security/zero-trust-security-2026-architecture-implementation-steps-common-pitfalls/)
- [Zero Trust Architecture Explained: SASE, SSE, ZTNA - Deepak Gupta](https://guptadeepak.com/research/zero-trust-architecture-explained/)
- [ISC2 Zero Trust Certification for Enterprise Enforcement](https://www.isc2.org/Insights/2026/04/zero-trust-certification-for-enterprise-enforcement)
- [Zero Trust for AI Workloads: Enterprise Guide 2026](https://beyondscale.tech/blog/zero-trust-ai-workloads-enterprise-guide)
