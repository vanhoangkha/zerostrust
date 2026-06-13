# Part 5: Zero Trust cho Cloud Native & Kubernetes

## Giới thiệu

Năm 2026, cloud-native security là **identity-centric và zero-trust by default**. Không còn là "nice-to-have" — mà là yêu cầu sống còn.

Con số nói lên tất cả: **89% tổ chức** đã gặp sự cố bảo mật Kubernetes trong năm qua, thường do **lateral movement** sau khi một pod bị xâm nhập ban đầu. Khi default Kubernetes configuration là flat network, exposed API server, và workloads có quá nhiều quyền truy cập — Zero Trust không phải lựa chọn, mà là **bắt buộc**.

## Tại sao Kubernetes cần Zero Trust?

### Vấn đề: "Inside" thay đổi liên tục

Perimeter security giả định "inside" an toàn hơn. Điều đó phá vỡ trong Kubernetes vì:
- Pods, services, identities liên tục thay đổi
- Network paths động, IP ephemeral
- Containers spinning up/down mọi lúc
- Default: **mọi pod nói chuyện tự do** với mọi pod khác

### Chuỗi tấn công điển hình

| Giai đoạn | Implicit Trust Risk | Zero Trust Mitigation |
|-----------|--------------------|-----------------------|
| **Initial Foothold** | Pod dùng shared default ServiceAccount | Unique, least-privilege ServiceAccount per pod |
| **Reconnaissance** | Attacker scan cluster network, hit K8s API | Default-deny NetworkPolicy block internal scanning |
| **Credential Theft** | Secrets mounted as env vars / clear-text | Short-lived tokens, encrypted vaulted secrets |
| **Lateral Movement** | Frontend pod → Backend → Database tự do | mTLS: mọi hop cần mutual authentication |
| **Privilege Escalation** | Wildcard RBAC (*) → ClusterAdmin | Explicit resource/verb mapping |

**Key insight:** Nguy hiểm không phải initial compromise — mà là **what happens next**. Quá nhiều RBAC permissions + unrestricted pod traffic + shared identities → small breach = full cluster control.

## Core Principles: Zero Trust trong Kubernetes

### 1. Verify Every Request

**Kubernetes API level:**
- API server checks authentication + RBAC authorization
- Admission controls validate trước khi persist object
- Mọi API change phải qua 3 gates: AuthN → AuthZ → Admission

**Service-to-service (Data plane):**
- CNI enforcing NetworkPolicies
- Service mesh (Istio) mTLS + workload identity
- Mỗi service phải **prove who they are**, không chỉ where they come from

### 2. Enforce Least Privilege

```yaml
# ❌ BAD: Overly broad permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: production
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]

---
# ✓ GOOD: Only what the app needs
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["app-config"]
    verbs: ["get"]
```

**Nguyên tắc:**
- Mỗi workload = một ServiceAccount riêng
- Roles scoped to namespace (không ClusterRoles trừ khi thực sự cần)
- Không wildcard verbs/resources
- Audit RBAC định kỳ cho wildcard (`*`) và unused ClusterRoleBindings

### 3. Assume Breach, Minimize Blast Radius

- Compromise namespace A ≠ access namespace B
- Compromise workload X ≠ access data stores it doesn't need
- Compromised CI job ≠ cluster admin

**Blast radius control = Network Policy + Identity Policy + Admission Guardrails** (cần cả 3).

### 4. Continuous Monitoring

Zero Trust ≠ "authenticate once, trust forever":
- Watch audit logs cho API access patterns
- Check policy drift
- Monitor runtime behavior cho abnormal process + network activity
- Phát hiện "valid pods doing invalid things"

## Key Security Controls

### 1. Network Segmentation: Default-Deny First

```yaml
# Bắt đầu bằng deny-all baseline
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

Sau đó add explicit allows:

```yaml
# Cho phép frontend → backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080

---
# Cho phép DNS egress cho tất cả pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

**Quan trọng:** Validate CNI plugin thực sự enforce policies! NetworkPolicy objects có thể tồn tại mà không enforcement nếu CNI không hỗ trợ (Cilium, Calico hỗ trợ; default kubenet thì không).

### 2. Service Mesh & mTLS

Service mesh (Istio 1.29, Linkerd) cung cấp:

**mTLS tự động:**
- Encrypt transit + workload identity cho service-to-service
- Auto-provision và rotate certificates
- Phasing incremental: PERMISSIVE mode → STRICT mode

**Authorization Policies (Istio):**

```yaml
# Chỉ cho phép frontend gọi backend's GET /api/orders
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/production/sa/frontend"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/api/orders*"]
```

**Migration path cho mTLS:**
1. Deploy Istio trong PERMISSIVE mode (accept cả plain text + mTLS)
2. Monitor traffic, verify tất cả services communicate qua mTLS
3. Switch sang STRICT mode (chỉ mTLS, reject plain text)

### 3. Workload Identity

Mỗi workload cần **cryptographic identity** unique:

```yaml
# Mỗi deployment = 1 ServiceAccount riêng
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  namespace: production
  annotations:
    # AWS: Map to IAM role
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/order-service
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      serviceAccountName: order-service  # Dedicated SA
      automountServiceAccountToken: false  # Disable if not needed
```

**Nguyên tắc:**
- Một ServiceAccount per Deployment/StatefulSet
- Disable automount token nếu không cần K8s API access
- Map to cloud IAM roles (EKS → IAM Roles for Service Accounts)
- Short-lived tokens (projected volumes) thay vì static tokens

### 4. Admission Controllers & Policy-as-Code

**OPA Gatekeeper / Kyverno** — enforce constraints at admission time:

```yaml
# Kyverno: Block privileged containers
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: Enforce
  rules:
    - name: deny-privileged
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Privileged containers are not allowed"
        pattern:
          spec:
            containers:
              - securityContext:
                  privileged: "false"
```

**Must-have admission policies:**
- Block privileged pods
- Require restricted Pod Security Standard
- Restrict image registries to approved sources
- Require resource limits
- Block hostPath mounts
- Require labels (ownership, environment)

### 5. Secrets Management

```yaml
# ❌ BAD: Secrets as environment variables
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password

# ✓ BETTER: External secrets with auto-rotation
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-secret
  data:
    - secretKey: password
      remoteRef:
        key: /production/db/password
```

**Best practices:**
- Vault / AWS Secrets Manager / GCP Secret Manager
- Short-lived credentials, auto-rotation
- Không hardcode secrets trong manifests
- Encrypt secrets at rest (EncryptionConfiguration)

## Zero Trust Kubernetes Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                            │
│                                                                 │
│  ┌──────────── Control Plane ─────────────────────────────────┐ │
│  │                                                            │ │
│  │  API Server ──▶ AuthN (OIDC) ──▶ AuthZ (RBAC) ──▶ Admit   │ │
│  │                                      │                     │ │
│  │                              OPA Gatekeeper / Kyverno      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌──────────── Data Plane ────────────────────────────────────┐ │
│  │                                                            │ │
│  │  ┌─────────┐    mTLS    ┌─────────┐    mTLS   ┌────────┐  │ │
│  │  │Frontend │◀──────────▶│ Backend │◀──────────▶│Database│  │ │
│  │  │  + Envoy│  (Istio)   │  + Envoy│           │        │  │ │
│  │  └─────────┘            └─────────┘           └────────┘  │ │
│  │       │                      │                     │       │ │
│  │       ▼                      ▼                     ▼       │ │
│  │  NetworkPolicy         AuthzPolicy           NetworkPolicy │ │
│  │  (Cilium/Calico)      (Istio/OPA)           (deny by def) │ │
│  │                                                            │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌──────────── Observability Layer ───────────────────────────┐ │
│  │  eBPF sensors │ Audit logs │ Flow logs │ Runtime detection  │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Implementation Roadmap: 5 Phases

### Phase 1: Assessment & Visibility (2-4 tuần)

| Task | Chi tiết |
|------|----------|
| Posture Inventory | Audit Namespaces, ServiceAccounts, RoleBindings |
| Traffic Mapping | eBPF/flow logs cho actual east-west traffic |
| Gap Analysis | Identify shared SAs, wildcard RBAC, no NetworkPolicies |

**Success metric:** 100% visibility into "shadow" traffic và RBAC sprawl

### Phase 2: Identity Hardening (4-6 tuần)

| Task | Chi tiết |
|------|----------|
| Split shared ServiceAccounts | 1 SA per deployment |
| Remove ClusterAdmin from humans | OIDC + SSO + short-lived admin access |
| Enforce OIDC/SSO | No static kubeconfig for production |

**Success metric:** Zero use of default ServiceAccount in production

### Phase 3: Network Micro-segmentation (6-8 tuần)

| Task | Chi tiết |
|------|----------|
| Deploy default-deny | Per namespace, start non-critical |
| Add explicit allows | Based on observed traffic from Phase 1 |
| Validate CNI enforcement | Test policies are actually blocking |

**Success metric:** Elimination of unplanned east-west traffic

### Phase 4: Runtime & Behavioral Baseline (4-6 tuần)

| Task | Chi tiết |
|------|----------|
| eBPF-based monitoring | Abnormal process execution, DNS tunneling |
| Alert setup | Exec into pods, unusual outbound calls |
| Incident response runbooks | Tie to K8s metadata |

**Success metric:** Immediate alerting on exec/unusual outbound

### Phase 5: Policy Automation (Ongoing)

| Task | Chi tiết |
|------|----------|
| OPA Gatekeeper / Kyverno | Block privileged, hostPath, unapproved registries |
| GitOps for security | NetworkPolicies + RBAC in Git, deploy via ArgoCD/Flux |
| Continuous validation | Periodic policy audit + drift detection |

**Success metric:** 0% deployment of privileged containers

## DevSecOps Integration

Zero Trust trong Kubernetes **phải** tích hợp CI/CD:

```
┌─────────────────────────────────────────────────────────┐
│                    CI/CD PIPELINE                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Code Push ──▶ Build ──▶ Test ──▶ Deploy ──▶ Monitor   │
│       │          │         │         │          │       │
│       ▼          ▼         ▼         ▼          ▼       │
│  • SAST      • Image    • Policy  • Admission  • eBPF  │
│  • Secrets    scan       validate   controllers • Audit │
│    detection • SBOM     (OPA/dry-  (enforce)    • Flow  │
│              • Sign      run)                    logs   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Nguyên tắc:**
- Security controls = code artifacts (versioned, reviewed, tested)
- Policies trong Git, deploy qua GitOps
- Shift-left: catch violations trong CI trước khi reach cluster
- Every security incident → new enforceable admission policy

## Cloud Provider Integration

### AWS EKS

- **IAM Roles for Service Accounts (IRSA):** Map K8s SA → IAM Role
- **Pod Identity Associations:** Simplified IRSA
- **VPC CNI NetworkPolicy support:** Native enforcement
- **GuardDuty EKS:** Runtime threat detection

### Azure AKS

- **Workload Identity Federation:** Map K8s SA → Managed Identity
- **Azure Policy (Gatekeeper):** Built-in admission policies
- **Calico NetworkPolicy:** Default supported
- **Microsoft Defender for Containers:** Runtime protection

### GCP GKE

- **Workload Identity:** Map K8s SA → GCP Service Account
- **Binary Authorization:** Enforce signed images only
- **Dataplane V2 (Cilium):** Native NetworkPolicy enforcement
- **Security Command Center:** Vulnerability + threat detection

## Common Failures & Solutions

| Failure | Hậu quả | Solution |
|---------|---------|----------|
| Zero trust without egress controls | Data exfiltration vẫn possible | FQDN filtering cho outbound |
| NetworkPolicies not enforced | False sense of security | Validate CNI enforcement actively |
| mTLS without authorization | Authentication ≠ access control | Pair mTLS + AuthorizationPolicy |
| RBAC sprawl without review | Privilege creep over time | Automated RBAC audits, ownership |
| Policies block without explanation | Teams bypass | Clear error messages, warn mode first |

## Best Practices 2026

| Practice | Action | Strategy |
|----------|--------|----------|
| Standardize Identity | 1 ServiceAccount per workload | Map SA → Deployment 1:1 |
| Policy-as-Code | GitOps for security | NetworkPolicies + RBAC in version control |
| Incident Hardening | Automated post-mortems | Every incident → new admission policy |
| Continuous Validation | eBPF-driven auditing | Kernel-level verify policy enforcement |
| Egress Governance | FQDN filtering | Limit outbound to specific, verified domains |
| Image Supply Chain | Signed images only | Binary Authorization + Cosign |

## Kubernetes Versions 2026

Kubernetes 1.35 và 1.36 (2026) cải thiện đáng kể Zero Trust capabilities:
- **Network Policy v2 APIs:** Richer policy expressions
- **Pod Security Admission:** Built-in enforcement (không cần PodSecurityPolicy)
- **Service Account Token Projection:** Default short-lived tokens
- **ValidatingAdmissionPolicy:** CEL-based policies native (không cần webhook)
- **Improved audit logging:** Richer context cho security events

## Tổng kết

### Zero Trust Kubernetes Checklist 2026

- [ ] Default-deny NetworkPolicy cho mọi production namespace
- [ ] Unique ServiceAccount per workload
- [ ] mTLS enabled (STRICT mode) cho service-to-service
- [ ] AuthorizationPolicy cho L7 access control
- [ ] OPA/Kyverno admission policies blocking privileged workloads
- [ ] Secrets management external (Vault/cloud secrets manager)
- [ ] eBPF runtime monitoring
- [ ] OIDC/SSO cho human access, no static kubeconfigs
- [ ] RBAC audit automation
- [ ] Security policies in Git (GitOps)
- [ ] Egress filtering (FQDN-based)
- [ ] Image signing + verification

Trong phần tiếp theo, chúng ta sẽ khám phá **Zero Trust cho AI Workloads** — lĩnh vực hoàn toàn mới trong 2026 với 6 trust boundaries chưa từng có.

---

## Nguồn tham khảo

- [Zero Trust in Kubernetes: Principles, Architecture & Best Practices - Groundcover](https://www.groundcover.com/learn/security/zero-trust-kubernetes)
- [Zero Trust Networking on Kubernetes: A Practical Guide - Vexxhost](https://vexxhost.com/blog/zero-trust-networking-kubernetes-practice/)
- [Zero-Trust on OKE: Secure Clusters With Terraform - CloudNativeNow](https://cloudnativenow.com/contributed-content/zero-trust-on-oke-how-to-actually-secure-your-clusters-with-terraform/)
- [Implementing Zero Trust in DevSecOps Workflow 2026 - Cloudaware](https://cloudaware.com/blog/zero-trust-devsecops/)
- [Embracing Zero Trust for Cloud-Native Security - DevOpsCon](https://devopscon.io/blog/devsecops/modernizing-threat-modeling/)
- [Seven Zero Trust Rules for Kubernetes - CNCF](https://www.cncf.io/blog/2022/11/04/seven-zero-trust-rules-for-kubernetes/)
- [The Sealos Security Path: Building a Zero-Trust Cloud-Native Platform](https://sealos.io/blog/the-sealos-security-path/)
- [Kubernetes Security Best Practices and Zero Trust Access - Appgate](https://www.appgate.com/blog/kubernetes-security-best-practices-zero-trust-access)
- [Istio Security Documentation](https://istio.io/latest/docs/concepts/security/)
