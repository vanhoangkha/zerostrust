# Part 6: Zero Trust cho AI Workloads

## Giới thiệu

Đây là chủ đề **hoàn toàn mới** trong 2026. Zero Trust truyền thống được thiết kế cho thế giới của human users, deterministic applications, và network perimeters. **AI workloads phá vỡ mọi giả định đó.**

Khi bạn deploy LLMs, AI agents, vector databases, và inference APIs trong enterprise environments, bạn tạo ra trust boundary problems mà NIST SP 800-207 **không hề đề cập**: non-human identities hoạt động tự chủ, instructions và data chia sẻ cùng một inference channel, và lateral movement ở tốc độ máy.

### Timeline quan trọng 2026

- **Tháng 1/2026:** Zscaler AI Security Report — 91% tăng trưởng AI activity, 100% hệ thống AI phân tích có lỗ hổng nghiêm trọng
- **Tháng 2/2026:** BSI (Đức) + ANSSI (Pháp) công bố Design Principles for LLM-based Systems with Zero Trust
- **Tháng 3/2026:** Microsoft phát hành **Zero Trust for AI (ZT4AI)** — framework, tools, reference architecture
- **Tháng 5/2026:** Microsoft Agent 365 GA — behavioral monitoring for agents
- **Tháng 6/2026:** Preprints.org — Zero Trust for AI Systems: Reference Architecture & Assurance Framework

## Tại sao Zero Trust truyền thống thất bại cho AI?

### 5 Lý do cốt lõi

| NIST 800-207 giả định | AI Workload thực tế | Hậu quả |
|---|---|---|
| Resources là identifiable assets | Model weights, vector DB, tokenizer files = protected resources mới | Không enumerate trong framework |
| Identities thuộc về humans hoặc deterministic service accounts | AI agents = category thứ 3: hold credentials, make API calls tự chủ, chain tools | NHI outnumber humans 45:1 đến 500:1 |
| Sessions có clear boundaries | Agent hoàn thành dozens of tasks/hour, mỗi task cần different data sources | Credential lifecycle mismatch |
| Data và instructions flow through separate channels | LLM: system prompt + user input + retrieved docs = **single string** at inference | Instruction-data collapse → prompt injection |
| Lateral movement cần thời gian | AI agent bị compromise → redirect tools + exfiltrate + instruct downstream agents **trong single inference cycle** | Lateral movement at machine speed |

### Instruction-Data Collapse — Vấn đề cốt lõi

```
Ứng dụng web truyền thống:
┌──────────────┐    ┌──────────────┐
│  User Input  │    │ System Logic │  ← SEPARATED by design
│  (data)      │    │ (code)       │     SQL injection = ở boundary
└──────────────┘    └──────────────┘

LLM Inference:
┌────────────────────────────────────────────┐
│  System Prompt + User Input + RAG Docs     │  ← ALL CONCATENATED
│  = Single inference string                 │     into ONE channel
│                                            │
│  Malicious instruction in RAG doc          │
│  = INDISTINGUISHABLE from system prompt    │
└────────────────────────────────────────────┘
```

Đây là **structural root cause** của indirect prompt injection (OWASP LLM01:2025).

### Non-Human Identities (NHI) at Scale

Theo NHI Reality Report 2026:
- Enterprise trung bình có **250,000+ non-human identities**
- NHIs outnumber humans tỷ lệ **45:1 đến 500:1**
- **97%** NHIs mang privileges **vượt quá** chức năng cần thiết
- **71%** NHI credentials không được rotated đúng timeframe
- **68%** IT security incidents 2026 liên quan machine identities

## Sáu Trust Boundaries trong AI Workloads

NIST SP 800-207 không define những boundaries này. Chúng là **unique to AI systems**:

```
┌────────────────────────────────────────────────────────────────┐
│                    AI TRUST BOUNDARIES                          │
│                                                                │
│  ┌──────┐  B1   ┌───────────┐  B2   ┌──────────┐             │
│  │ User │──────▶│ Inference │──────▶│  Model   │             │
│  └──────┘       │    API    │       │ Weights  │             │
│                 └─────┬─────┘       └──────────┘             │
│                       │                                       │
│                       │ B3                                     │
│                       ▼                                       │
│                 ┌───────────┐  B4   ┌──────────┐             │
│                 │   Tools   │──────▶│   Data   │             │
│                 │  Plugins  │       │ Sources  │             │
│                 └─────┬─────┘       └──────────┘             │
│                       │                                       │
│                       │ B5                                     │
│                       ▼                                       │
│                 ┌───────────┐  B6   ┌──────────┐             │
│                 │   Agent   │──────▶│ External │             │
│                 │  (other)  │       │ Services │             │
│                 └───────────┘       └──────────┘             │
└────────────────────────────────────────────────────────────────┘
```

### Boundary 1: User → Inference API

- Mọi request đến inference endpoint phải **authenticated** (không chỉ network-trusted)
- Authorization theo user's identity context
- Rate-limited + logged
- Gateway layer với mTLS hoặc OAuth 2.0 — **không chỉ VPC network controls**

### Boundary 2: Inference Service → Model Weights

- Model weights = **protected asset** (IP cực kỳ giá trị)
- Encryption at rest với customer-managed keys
- Access load/modify weights phải audited với tamper-evident logs
- **Confidential computing enclaves** (AMD SEV, Intel TDX): weights decrypt chỉ trong processor's secure memory
- Microsoft ZT4AI: confidential computing = core architectural recommendation

### Boundary 3: Model → Tools/Plugins

- Tool calls từ LLMs = **primary attack surface**
- Agent invoke tools phải dùng **scoped, time-limited credentials**
- Tool access restricted to **defined allow-list** (không open access)
- Tool definitions phải **integrity-verified** (cryptographic pinning)
- Compromised MCP server → inject malicious tool descriptions → redirect model behavior

### Boundary 4: Tools → Data Sources

- Khi agent tool query DB/vector store/external API:
- Phải enforce **requesting human user's authorization context**
- **Không dùng shared agent identity** cho data access
- OAuth token delegation: user → agent → tool
- Ngăn agent retrieve data mà user không có quyền access trực tiếp

### Boundary 5: Agent → Agent (Multi-Agent)

- Inter-agent messages = **potential injection vectors**
- Microsoft ZT4AI: **Agent Identity Token (AIT)** — encodes:
  - Agent's identity
  - System prompt hash
  - Knowledge base version
- Receiving agent verifies AIT trước khi process message
- System prompt deviated from baseline → **connection terminates**
- Prevents prompt-based lateral movement through agent mesh

### Boundary 6: Pipeline → External Services

- AI pipelines connect external APIs, email, cloud storage
- **Egress controls:** explicit allow-list policies (không open internet)
- Log + inspect outbound traffic cho data exfiltration/C2 signs

## Microsoft Zero Trust for AI (ZT4AI) Framework

### Ba nguyên tắc ZT áp dụng cho AI

| Nguyên tắc | Áp dụng cho AI |
|---|---|
| **Verify explicitly** | Continuously evaluate identity + behavior của AI agents, workloads, users |
| **Apply least privilege** | Restrict access to models, prompts, plugins, data sources → chỉ what's needed |
| **Assume breach** | Design AI systems resilient to prompt injection, data poisoning, lateral movement |

### ZT4AI Tools & Guidance (March 2026)

1. **AI Pillar trong Zero Trust Workshop** — 700 security controls, 116 logical groups, 33 functional swim lanes
2. **Zero Trust Assessment** — automated evaluation, AI pillar coming summer 2026
3. **Reference Architecture for AI** — policy-driven access, continuous verification, monitoring, governance
4. **Patterns & Practices:**
   - Threat modeling for AI
   - AI observability
   - Securing agentic systems
   - Robust safety engineering
   - Defense-in-depth for indirect prompt injection (XPIA)

### AI Pillar evaluates:

- Cách tổ chức **secure AI access** và **agent identities**
- **Protect sensitive data** used by/generated through AI
- **Monitor AI usage** và behavior across enterprise
- **Govern AI responsibly** aligned with risk/compliance

## Identity cho AI Agents

### Requirements (Cisco Zero Trust for Agentic AI, March 2026)

1. **Know every agent:** Inventory tất cả deployed agents, models used, tools callable, human owner
2. **Authorize every action:** Each agent = NHI mapped to responsible human owner, least-privilege per task
3. **Adapt to risk real-time:** Behavioral monitoring, anomalous patterns → automated response

### Implementation Components

#### Machine Identity Management
```
SPIFFE/SPIRE cho AI Agents:
- Issue short-lived X.509 certificates cho agent processes
- Certificates expire in HOURS, not years
- Rotation automated, no manual intervention
- spiffe://ai-trust-domain/agent/summarizer-v2
```

#### Just-In-Time Credentials
```
Agent receives:
- OAuth 2.0 access token scoped to specific tools + data sources
- Token expires at task completion
- No standing API keys with broad access
- RFC 8693 Token Exchange: scoped tokens on behalf of users
```

#### Agent Identity Inventory
```yaml
# Agent Registry Entry
agent_id: summarizer-agent-v2
owner: data-team@company.com
model: gpt-4-turbo
tools_allowed:
  - document-retrieval (read-only)
  - text-summarization
tools_denied:
  - code-execution
  - email-send
  - database-write
max_data_access: classification <= INTERNAL
last_activity: 2026-06-13T10:30:00Z
credential_rotation: every 4 hours
```

## Least Privilege cho AI: Ba Scopes

### 1. Tool Scope

```
Document summarization agent:
  ✓ document-retrieval (read)
  ✓ text-summarization
  ✗ code-execution
  ✗ web-search
  ✗ database-write
  ✗ email-send
  
→ Default: tool NOT on allow-list = BLOCKED
→ Addresses OWASP LLM06:2025 (Excessive Agency)
```

### 2. Data Scope

- Agent queries vector DB **under user's authorization context**
- Row-level + column-level access controls
- Agent presents **user's delegated credential** (không shared service identity)
- Metadata filtering at query time: `access_level <= user.clearance`

### 3. Action Scope — Human-in-the-Loop

**High-impact actions cần human approval:**
- Send emails
- Modify files
- Execute code
- Call external APIs with write access
- Access payment/HR systems

**CSA Agentic Trust Framework maturity:**
- **Intern tier:** Read-only agents
- **Associate tier:** Limited-write với approval gates
- **Senior tier:** Autonomous write access (highest trust)

## Network Segmentation cho AI Infrastructure

### Bốn Trust Zones

```
┌─────────────────────────────────────────────────────────────┐
│  ZONE 1: User Access Layer                                  │
│  - Centralized IdP authentication                           │
│  - OAuth 2.0 short-lived tokens                             │
│  - No direct user access to inference/models                │
├─────────────────────────────────────────────────────────────┤
│  ZONE 2: Inference & Orchestration Layer                    │
│  - LLM inference endpoints                                  │
│  - Agent orchestrators, LLM gateways                        │
│  - Isolated from other app workloads                        │
│  - Outbound tool calls → authorized tool broker only        │
├─────────────────────────────────────────────────────────────┤
│  ZONE 3: Tools & Data Layer                                 │
│  - MCP servers, vector databases, API connectors            │
│  - Enforce requesting user's auth context                   │
│  - No standing broad-access service identities              │
│  - Vector retrieval includes ACL metadata filters           │
├─────────────────────────────────────────────────────────────┤
│  ZONE 4: Model & Training Layer                             │
│  - Model weights, registries, training data                 │
│  - COMPLETELY ISOLATED from inference                        │
│  - Model promotion → integrity verification (hash + sign)   │
│  - No inference-layer access to training data               │
└─────────────────────────────────────────────────────────────┘

All cross-zone traffic: authenticated + authorized + logged
No implicit trust based on network location
```

### Segmentation Requirements

| Component | Isolation Requirement |
|-----------|----------------------|
| Inference endpoints | Dedicated segment, allow-list ingress, no lateral to DBs |
| Vector databases | Not directly reachable from general apps, only via RAG/inference layer |
| MCP servers | Own trust zone, only agent orchestration communicates |
| Training environments | Separate segment, no direct connectivity to inference |

## Data Plane Controls: RAG Security

### Prompt-Level Access Control

- System prompts = **access-controlled configuration assets** (không phải open text)
- Store in secret management (Vault, AWS Secrets Manager)
- Audit access, version-control changes with approval gates
- Alert on queries attempting to extract system prompt content

### Retrieval Authorization in RAG

```python
# ❌ BAD: No access control on retrieval
results = vector_store.query(
    embedding=query_embedding,
    top_k=10
)

# ✓ GOOD: Enforce user's access level
results = vector_store.query(
    embedding=query_embedding,
    top_k=10,
    filter={
        "access_level": {"$lte": user.clearance_level},
        "department": {"$in": user.departments},
        "classification": {"$lte": "INTERNAL"}
    }
)
```

**Pre-ingestion validation:** Validate documents cho prompt injection patterns **trước khi** embed vào vector store. Remediation sau khi embedded = significantly harder.

### Output Sanitization (Trust Gate)

Trước khi AI output → another agent/system/user:
- Check attempts to override downstream instructions
- Detect data exfiltration patterns (encoded data, unexpected URLs)
- Validate format against expected output schema
- **Sit between EVERY agent boundary** trong multi-agent systems

## Zero Trust cho Multi-Agent Systems

### Ba Failure Patterns

**1. Trust Inheritance Without Verification**
- Agent A authorized → access sensitive data
- Agent A calls Agent B → data flows with A's implicit trust
- Agent B compromised → uses inherited trust for unauthorized actions

**2. Blast Radius Expansion Through Tool Chains**
- Agent compromised ở step 2 → retains access to tools/data of steps 2-10
- Blast radius = combined permissions of steps 2 through 10

**3. Prompt Injection via Inter-Agent Messages**
- Upstream agent output → downstream agent input
- Attacker influences upstream → injects instructions downstream executes

### Controls for Multi-Agent Zero Trust

```
1. Re-authenticate at each agent boundary
   (không inherit trust from calling agent)
   
2. Re-scope permissions at each handoff
   (each agent gets ONLY what it needs for ITS step)
   
3. Tag messages with provenance metadata
   (distinguish instructions vs processed data)
   
4. Validate output before forwarding
   (sanitization gate between EVERY agent)
   
5. Blast radius containment
   (each agent's max reachable scope = independent)
```

## Continuous Verification: Behavioral Monitoring

### Baselines per Agent

Establish baselines cho mỗi agent:
- Expected tool call sequences
- Typical data access volumes
- Normal output characteristics
- Expected external API targets

### Anomaly Indicators

| Signal | Ý nghĩa |
|--------|---------|
| Tool calls outside allow-list | Agent bị hijacked |
| Data retrieval volumes >> baseline | Potential exfiltration |
| Requests to unexpected external endpoints | C2 communication |
| Queries extracting system prompt | Prompt theft attempt |
| Recursive self-modification | Agent autonomy escape |
| Elevated tool call error rates | Active probing/injection |

### CSA Agentic Trust Framework

60% of architecture addresses **resilience after prevention fails**: behavioral monitoring + segmentation + incident response.

## 90-Day Implementation Roadmap

### Days 1-30: Inventory & Baseline

**Visibility before controls. Cannot enforce ZT on undiscovered assets.**

- [ ] Complete agent + model inventory (models, agents, tools, data sources)
- [ ] Map all NHIs (API keys, service accounts, OAuth clients, model registry tokens)
- [ ] Document scope + last-rotation date cho mỗi credential
- [ ] Identify externally-exposed inference endpoints
- [ ] Begin logging agent tool calls, data volumes, external API targets
- [ ] Establish behavioral baselines

### Days 31-60: Identity & Access Hardening

- [ ] Replace long-lived API keys → machine identities + short-lived OAuth tokens
- [ ] Implement per-agent tool allow-lists
- [ ] Add human-in-the-loop gates cho high-impact actions
- [ ] Deploy retrieval authorization in RAG (access control metadata filtering)
- [ ] Cryptographically pin MCP tool definitions
- [ ] Alert on definition changes outside approved process

### Days 61-90: Network Segmentation & Monitoring

- [ ] Implement 4-zone network segmentation (user/inference/tools+data/model+training)
- [ ] Deploy output sanitization between agent boundaries
- [ ] Enable behavioral anomaly monitoring
- [ ] Implement system prompt integrity verification (hash baseline, alert on deviation)
- [ ] Conduct tabletop exercise: multi-agent compromise scenario
- [ ] Validate blast radius containment

### Quick Wins (Immediate Risk Reduction)

1. **Rotate all long-lived AI agent API keys** → short-lived tokens
2. **Add authentication** to inference endpoints using only network controls
3. **Restrict agent tool access** to explicitly defined allow-lists

## BSI/ANSSI Design Principles (2026)

German BSI + French ANSSI jointly published 6 principles cho LLM systems:

1. **Authentication & authorization per interaction** (không per session)
2. **Input & output restrictions** (validate both directions)
3. **Sandboxing** (isolate execution environments)
4. **Monitoring & control** (continuous behavioral observation)
5. **Threat intelligence integration** (feed into policy engine)
6. **Stakeholder awareness** (business owners understand AI risks)

## Tổng kết

### Zero Trust for AI ≠ Incremental Extension

Nó yêu cầu **rethinking**:
- **Identity model:** NHIs, JIT credentials, agent identity verification, AIT
- **Data plane:** Instruction-data collapse, retrieval authorization, prompt-level access
- **Monitoring:** Behavioral baselines, per-agent anomaly detection, inter-agent message validation

### Key Framework Convergence 2026

| Framework | Contribution |
|-----------|-------------|
| Microsoft ZT4AI | Tools, workshop, reference architecture, patterns |
| BSI/ANSSI Joint Publication | 6 design principles cho LLM systems |
| CSA Agentic Trust Framework | Agent maturity levels, resilience architecture |
| OWASP LLM Top 10 2025 | Attack taxonomy (prompt injection, excessive agency) |
| Cisco Zero Trust for Agentic AI | Agent identity governance requirements |

Tất cả đều converge cùng conclusion: **Zero Trust truyền thống là necessary nhưng NOT sufficient cho AI workloads. Cần AI-specific controls layered on top.**

👉 Đọc tiếp [Part 7: Lộ trình triển khai Zero Trust cho doanh nghiệp](part7-lo-trinh-trien-khai-zero-trust.md) — tổng hợp tất cả thành roadmap triển khai end-to-end, từ assessment đến production.

---

## Nguồn tham khảo

- [Zero Trust for AI Workloads: Enterprise Guide 2026 - BeyondScale](https://beyondscale.tech/blog/zero-trust-ai-workloads-enterprise-guide)
- [New tools and guidance: Announcing Zero Trust for AI - Microsoft](https://www.microsoft.com/en-us/security/blog/2026/03/19/new-tools-and-guidance-announcing-zero-trust-for-ai/)
- [Zero Trust AI: The Enterprise Security Model for AI Systems in 2026 - HyScaler](https://hyscaler.com/insights/zero-trust-ai-framework/)
- [Zscaler 2026 AI Security Report](https://www.zscaler.com/press/zscaler-2026-ai-threat-report-91-year-over-year-surge-ai-activity-creates-growing-oversight)
- [Why trust will define the $20 trillion AI opportunity - WEF](https://www.weforum.org/stories/2026/01/ai-the-next-innovation-frontier-trust-define-20-trillion-opportunity/)
- [Generative AI and Zero Trust Security - CSA](https://cloudsecurityalliance.org/articles/how-generative-ai-is-reshaping-zero-trust-security)
- [Zero Trust for AI Systems: Reference Architecture - Preprints.org](https://www.preprints.org/manuscript/202602.0085/v1)
- [BSI/ANSSI Design Principles for LLM-based Systems with Zero Trust](https://www.bsi.bund.de/SharedDocs/Downloads/EN/BSI/Publications/ANSSI-BSI-joint-releases/LLM-based_Systems_Zero_Trust.pdf)
- [Cisco Zero Trust for Agentic AI White Paper](https://www.cisco.com/c/en/us/solutions/collateral/artificial-intelligence/security/zero-trust-agentic-ai-wp.html)
