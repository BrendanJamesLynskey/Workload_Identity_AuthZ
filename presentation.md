# Workload Identity & Service-Mesh AuthZ

**SPIFFE/SPIRE · Istio · Linkerd · Cilium · Cloud-native federation**

*How services prove who they are to other services — without shared secrets, without long-lived keys, and without an operator in the loop.*

```
attest -> issue SVID -> mTLS handshake -> policy decision
```

Attest · Issue · Verify · Authorise

---

## Table of Contents

1. [Topics](#slide-01--topics)
2. [The Workload-Identity Problem](#slide-02--the-workload-identity-problem)
3. [SPIFFE — The Specification](#slide-03--spiffe--the-specification)
4. [SPIRE — The Reference Implementation](#slide-04--spire--the-reference-implementation)
5. [Workload Attestation](#slide-05--workload-attestation--how-spire-knows-what-you-are)
6. [SPIFFE Federation](#slide-06--spiffe-federation--across-trust-domains)
7. [Service Mesh — What It Buys You for AuthZ](#slide-07--service-mesh--what-it-buys-you-for-authz)
8. [Istio AuthorizationPolicy](#slide-08--istio-authorizationpolicy--patterns--pitfalls)
9. [Linkerd](#slide-09--linkerd--server-serverauthorization--httproute)
10. [Cilium + Tetragon](#slide-10--cilium--ebpf-native-l4l7--tetragon)
11. [mTLS at Scale](#slide-11--mtls-at-scale--issuance-rotation-observability)
12. [Cloud-Native Workload Identity](#slide-12--cloud-native-workload-identity)
13. [K8s ServiceAccountTokenVolumeProjection](#slide-13--k8s-serviceaccounttokenvolumeprojection)
14. [Service Mesh + JWT — End-User Identity at the Sidecar](#slide-14--service-mesh--jwt--end-user-identity-at-the-sidecar)
15. [Workload Identity for AI Agents & MCP Servers](#slide-15--workload-identity-for-ai-agents--mcp-servers)
16. [Choosing a Stack](#slide-16--choosing-a-stack--mesh-vs-spire-vs-cloud-native)
17. [Migration Patterns](#slide-17--migration-patterns)
18. [Production Gotchas](#slide-18--production-gotchas)
19. [Summary & References](#slide-19--summary--references)

---

## Slide 01 — Topics

### The workload-identity problem
- Why service-to-service is fundamentally different from user-to-service
- The crisis of long-lived API keys
- What "identity" means for a process / pod / Lambda

### SPIFFE & SPIRE
- SPIFFE ID and the SVID — X.509 and JWT flavours
- Workload attestation
- Federation — SPIFFE bundles across trust domains

### Service mesh AuthZ
- Mesh primer — what it gives you for AuthZ
- Istio AuthorizationPolicy — patterns & pitfalls
- Linkerd authorisation
- Cilium L4/L7 + Tetragon
- mTLS rotation at scale
- Mesh + JWT (Envoy OIDC at L7)

### Cloud-native & operational
- IRSA, GCP Workload Identity, Azure WI, WIF
- K8s ServiceAccountTokenVolume
- Workload identity for AI agents / MCP
- Choosing a stack · migration patterns · gotchas

---

## Slide 02 — The Workload-Identity Problem

Every modern system has a sea of services calling other services. They authenticate using secrets — tokens, API keys, certificates. **Where do those secrets come from, where are they stored, and what happens when one leaks?**

### The "shared secret in a config file" era

- API keys committed to git, copy-pasted between Slack channels.
- "Service account" passwords typed once at install and never rotated.
- Static AWS access keys baked into AMIs.
- Every breach post-mortem: *"The credential had been valid for ~ years."*

### The user-AuthN solution doesn't translate

- OAuth assumes a human in the loop to consent.
- Passwords / passkeys assume a person with a device.
- Services have *none* of those — they boot, attempt a connection, and need to prove themselves immediately.

### What a workload-identity system gives you

- **Cryptographic identity** bound to *what* the workload is, not *where* a secret was placed.
- **Short-lived** credentials — minutes, not years.
- **Automatic rotation** with no application-side change.
- **Verifiable** — receiver checks the identity against a trust anchor.
- **Universal** — same model across containers, VMs, serverless, on-prem, multi-cloud.

### Two camps of solutions

- **Cloud-native** — IRSA, GCP Workload Identity, Azure Workload Identity, WIF. Tied to the cloud's identity service.
- **Standards-based** — SPIFFE/SPIRE — vendor-neutral, multi-cloud, on-prem.
- Most production stacks combine both: SPIRE issues SPIFFE IDs; SPIRE federates to AWS/GCP via OIDC.

---

## Slide 03 — SPIFFE — The Specification

**SPIFFE** = Secure Production Identity Framework For Everyone. CNCF graduated 2022. A specification, not a product.

### SPIFFE ID — the identifier

```
spiffe://acme.com/billing/payments-svc

      └── trust domain ──┘└── workload path ──┘
```

- URI format. Trust domain = organisational boundary; path = workload identifier.
- Path is yours to design — by team / service / environment / pod.
- Two workloads with the same SPIFFE ID are considered the same identity.

### SVID — SPIFFE Verifiable Identity Document

- The cryptographic credential a workload presents.
- Two flavours: **X.509 SVID** (a cert) and **JWT SVID** (a JWT).
- Short-lived — typically 5 min to 1 hour.
- Always tied back to a SPIFFE ID via `SAN: URI:spiffe://...` (X.509) or `sub` (JWT).

### An X.509 SVID, decoded

```
Certificate:
  Subject: O=SPIRE
  X509v3 Subject Alternative Name:
    URI:spiffe://acme.com/billing/payments-svc
  Validity:
    Not Before: 2026-05-06T09:00:00Z
    Not After:  2026-05-06T10:00:00Z
  Public Key: ECDSA P-256 …
  Signed by: spiffe://acme.com (intermediate CA)
```

### A JWT SVID, decoded

```json
{
  "iss": "spiffe://acme.com",
  "sub": "spiffe://acme.com/billing/payments-svc",
  "aud": ["spiffe://acme.com/billing/ledger-svc"],
  "exp": 1800003600,
  "iat": 1800000000
}
```

JWT SVIDs are sender-constrained by audience.

---

## Slide 04 — SPIRE — The Reference Implementation

**SPIRE** = SPIFFE Runtime Environment. Server (central) + Agent (one per node). Together they issue SVIDs after attesting workloads.

```
[ SPIRE Server ] ◀── node attestation ── [ SPIRE Agent ] ◀── Workload API (UDS) ── [ Workload ]
                                            (one per node)                          (Pod / Lambda / VM)

1. Agent attests itself (k8s_psat / aws_iid / azure_msi / sshpop / x509pop)
   → Agent is now trusted by the Server, gets a node-level identity.

2. Workload connects to local Agent over Unix Domain Socket.
   → Agent inspects PID, k8s namespace+SA, container image hash, etc. (workload attestation)

3. Agent matches selectors to a registration entry, asks Server for an SVID.
   → Server signs SVID with the trust-domain CA, returns to Agent.

4. Agent hands the SVID to the workload (and rotates it before expiry).
```

### Key property — no secrets in app code

The workload has no shared key with the Server. The Agent's attestation chain proves the workload is legitimate; the SVID falls out as a by-product.

---

## Slide 05 — Workload Attestation — How SPIRE Knows What You Are

The attestation chain is the heart of SPIRE.

### Node attestation plugins

- `k8s_psat` — Kubernetes projected SA token, validated against cluster's OIDC issuer.
- `aws_iid` — AWS instance identity document signed by EC2.
- `aws_iam` — STS GetCallerIdentity proves the EC2 role.
- `gcp_iit` / `azure_msi` — equivalents.
- `x509pop` / `sshpop` — proof-of-possession of pre-issued credentials.
- `tpm` — TPM EK/AK chains for bare metal.

### Workload attestation plugins

- **k8s** — namespace, SA, pod label, container image SHA.
- **unix** — UID, GID, parent process binary path + checksum.
- **docker** — container labels, image digest.
- **systemd** — cgroup name, unit name.

### A registration entry — the policy

```
spire-server entry create \
  -spiffeID  spiffe://acme.com/billing/payments-svc \
  -parentID  spiffe://acme.com/spire/agent/k8s_psat/cluster/abc \
  -selector  k8s:ns:billing \
  -selector  k8s:sa:payments-svc \
  -selector  k8s:container-image-sha:sha256:e1b7… \
  -ttl       300
```

The agent only mints this SPIFFE ID for a workload that *matches every selector*.

### The selector trap

Loose selectors give every Pod in a namespace the same identity. Tight selectors (image SHA) break on every deploy. Sweet spot: namespace + service account + image tag with admission policy enforcing valid image registries.

---

## Slide 06 — SPIFFE Federation — Across Trust Domains

### The problem

You have `spiffe://acme.com` for production. Your acquired company runs `spiffe://oldco.com`. A service in acme needs to call a service in oldco. Each trust domain has its own CA.

### The mechanism — Trust Bundles

- Each SPIRE Server publishes a *trust bundle*: the public keys / CA certs of the trust domain.
- Federation = exchanging trust bundles, statically or via the SPIFFE Federation API.
- Once trust-bundles are exchanged, an X.509 SVID from `oldco` validates against acme's view, and vice versa.
- Authorisation policy still has the final say — federation says "I trust the issuer", not "I trust this caller".

### The Federation API

```
GET /federation/spiffe/v1/bundle

{
  "spiffe_sequence": 1234,
  "spiffe_refresh_hint": 60,
  "keys": [
    { "kty": "EC", "use": "x509-svid", "x5c": ["MIIB…"] },
    { "kty": "EC", "use": "jwt-svid",  "kid": "abc1", "crv": "P-256", "x":"…", "y":"…" }
  ]
}
```

### Use cases for federation

- Multi-cloud (one SPIRE per cloud, federated).
- Multi-cluster Kubernetes — one trust domain per cluster.
- Federation with cloud-native identity (AWS as a federated trust source via OIDC).
- B2B service-to-service across organisations.

---

## Slide 07 — Service Mesh — What It Buys You for AuthZ

A service mesh injects a sidecar (or, in newer architectures, an ambient agent) into every pod's network path. That sidecar can do three things app code traditionally had to: identity, encryption, policy.

```
[Pod A: app] ── localhost ── [sidecar A] ──── mTLS (SPIFFE-ID certs both ways) ──── [sidecar B] ── localhost ── [Pod B: app]
                                              policy decision: AuthorizationPolicy / NetworkPolicy / EndpointPolicy
```

App code sees plain HTTP to localhost. Sidecar handles identity + crypto + policy. *App code didn't change.*

### Identity
Sidecar holds the workload's SVID / cert. Other pod sees the verified identity, not an IP.

### Encryption
mTLS for free, between every pair of pods. Rotated by the mesh, not the app.

### Policy
Policies expressed as Kubernetes CRDs, applied at the sidecar — outside the app.

---

## Slide 08 — Istio AuthorizationPolicy — Patterns & Pitfalls

Istio's `AuthorizationPolicy` CRD is evaluated by the Envoy sidecar (or ztunnel in ambient mode). Supports L4 and L7 rules.

### A typical L7 rule

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata: { name: ledger-allow-payments, namespace: billing }
spec:
  selector:
    matchLabels: { app: ledger-svc }
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/billing/sa/payments-svc"]
      to:
        - operation:
            methods: ["GET","POST"]
            paths:   ["/v1/postings/*"]
      when:
        - key:    request.auth.claims[scope]
          values: ["postings:write"]
        - key:    request.headers[x-tenant-id]
          notValues: [""]
```

### The four important things

- **action** — `ALLOW`, `DENY`, `AUDIT`, or `CUSTOM` (delegate to ext_authz).
- **selector** — which workloads this attaches to. Empty = whole namespace.
- **rules.from.source.principals** — *SPIFFE-style identities* (`cluster.local/ns/X/sa/Y`).
- **when** — predicates on JWT claims, headers, source IP, request properties.

### Pitfalls in production

- **Default-allow** — without an explicit DENY-all + ALLOW catalogue, anything can call anything.
- **Authn vs authz mix-up** — an AuthorizationPolicy without a paired `RequestAuthentication` evaluates JWT claims against an unverified token.
- **Action precedence** — DENY wins over ALLOW; CUSTOM (ext_authz) runs first.
- **Selectors that match too much** — a label collision can extend a payments policy to a non-payments service.

---

## Slide 09 — Linkerd — Server, ServerAuthorization & HTTPRoute

Linkerd's authorisation model is intentionally smaller than Istio's — fewer CRDs, simpler defaults, gateway-API-aligned.

### The three CRDs

```yaml
# 1. Server — selects ports on workloads to protect
apiVersion: policy.linkerd.io/v1beta3
kind: Server
metadata: { name: ledger, namespace: billing }
spec:
  podSelector: { matchLabels: { app: ledger-svc } }
  port: 8080
  proxyProtocol: HTTP/2

---
# 2. AuthorizationPolicy — bind a target to required auth
apiVersion: policy.linkerd.io/v1alpha1
kind: AuthorizationPolicy
metadata: { name: ledger-from-payments, namespace: billing }
spec:
  targetRef: { group: policy.linkerd.io, kind: Server, name: ledger }
  requiredAuthenticationRefs:
    - { group: policy.linkerd.io, kind: MeshTLSAuthentication, name: payments-svc-id }

---
# 3. MeshTLSAuthentication — who is allowed
apiVersion: policy.linkerd.io/v1alpha1
kind: MeshTLSAuthentication
metadata: { name: payments-svc-id, namespace: billing }
spec:
  identities:
    - "payments-svc.billing.serviceaccount.identity.linkerd.cluster.local"
```

### Why split into three?

Composing them lets you say "*port 8080 of ledger requires either an mTLS identity from list X **or** a JWT with claim Y*".

### Default deny

Linkerd 2.12+ defaults to *"ports without a Server are open; ports with a Server require an explicit policy"*. Add a Server and you've turned on the lock.

### Linkerd vs Istio

Linkerd is lighter and more opinionated; Istio more powerful and more complex. Rule of thumb: Linkerd unless you specifically need Istio's L7 features.

---

## Slide 10 — Cilium — eBPF-Native L4/L7 + Tetragon

### Cilium in two sentences

An **eBPF**-based CNI for Kubernetes. Network policy, observability, encryption and L7 inspection are enforced by eBPF programs in the kernel — no per-pod sidecar, often lower overhead.

### CiliumNetworkPolicy at L7

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: { name: payments-to-ledger, namespace: billing }
spec:
  endpointSelector:
    matchLabels: { app: ledger-svc }
  ingress:
    - fromEndpoints:
        - matchLabels: { app: payments-svc }
      toPorts:
        - ports: [{ port: "8080", protocol: TCP }]
          rules:
            http:
              - method: "GET"
                path:   "/v1/postings/.*"
              - method: "POST"
                path:   "/v1/postings"
```

### Where Cilium shines

- Sidecar-free meshes — Cilium's "Service Mesh" mode (and ambient mesh) puts policy/encryption in the host kernel.
- Kernel-level observability — **Hubble** shows L3-L7 traffic with identity attached.
- Identity-aware NetworkPolicy — pods identified by labels, not IPs.
- WireGuard or mTLS encryption between nodes.

### Tetragon — runtime security

- Sister project. eBPF-based runtime visibility: every `execve`, `connect`, file open.
- Policies can *kill* a process in-kernel on detection.
- Less an "AuthZ" tool, more a "post-AuthZ enforcement on what the workload actually does" tool.

### Watch out

Cilium's L7 inspection only works for protocols it understands (HTTP/1.1, HTTP/2, gRPC, Kafka, DNS). For anything custom you're back to L4 + a service mesh sidecar.

---

## Slide 11 — mTLS at Scale — Issuance, Rotation, Observability

### The bits that hurt

- **Issuance** — every workload needs a fresh cert when it boots, ideally without ever holding a long-lived CA key.
- **Rotation** — short-lived certs (5–60 min) re-rolled before expiry, automatically.
- **Trust bundle distribution** — every verifier needs the latest CA bundle even as you rotate roots.
- **CA root rotation** — rare but required, and many setups never rehearse it.

### Who issues, in practice

| System | Who issues |
|---|---|
| Istio (in-mesh) | Istiod / cert-manager+istio-csr / SPIRE federated |
| Linkerd | Linkerd's own identity component / cert-manager |
| Cilium service mesh | Cilium agent, optionally with cert-manager |
| SPIRE-issued mesh | SPIRE Server |
| Cloud-managed (App Mesh, Service Mesh) | Cloud's CA service |

### cert-manager + Trust Manager

- Kubernetes-native CA / certificate operator.
- `Issuer` / `ClusterIssuer` CRDs for ACME, Vault, AWS PCA, SPIFFE.
- **Trust Manager** distributes CA bundles into ConfigMaps so meshes / proxies pick them up automatically.

### Things to monitor

- Cert expiry distribution per workload (alert when < 50 % of intended TTL).
- Re-issuance failure rate.
- Mismatched trust bundles between regions / clusters.
- mTLS handshake failure rate at the sidecar / gateway.

### CA root rotation

Rehearse it before you need it. Pre-distribute the new root for ≥ 1 cert TTL before switching issuance, then keep the old root in trust bundles for ≥ 1 cert TTL after.

---

## Slide 12 — Cloud-Native Workload Identity

| Cloud | Mechanism | How it binds |
|---|---|---|
| **AWS** — IRSA | EKS pod's projected SA token has `aud=sts.amazonaws.com` | STS trusts the cluster's OIDC issuer; pod calls `AssumeRoleWithWebIdentity`. |
| **AWS** — EKS Pod Identity | Newer alternative; agent on every node | Agent intermediates; pod calls a local IMDSv2-like endpoint. |
| **GCP** — Workload Identity | Bind GCP SA to K8s SA | GKE metadata server intermediates; pod gets GCP creds via metadata calls. |
| **GCP** — Workload Identity Federation | Outside-GCP workloads | External OIDC IdP → STS-style token exchange → short-lived GCP creds. |
| **Azure** — Workload Identity | K8s SA federated with Entra app | SA token exchanged via OIDC for an Entra access token. |
| **Azure** — Managed Identity | VMs / Container Apps / Functions | Local IMDS endpoint at `169.254.169.254` mints tokens. |

### The common shape

An OIDC token signed by the workload-substrate (cluster, cloud) is exchanged for short-lived API credentials at the cloud's STS. **No long-lived secret, no shared key, no manual rotation.**

### Where it differs from SPIFFE

Cloud workload identity authenticates the workload *to the cloud's APIs*. SPIFFE authenticates workloads *to each other*. They compose: SPIRE can attest using IRSA, and SPIFFE workloads can federate to AWS via WIF.

---

## Slide 13 — K8s ServiceAccountTokenVolumeProjection

The Kubernetes-native way to get a short-lived OIDC-style token into a pod. The substrate everything cloud-native workload identity (IRSA, GCP WI, Azure WI) and SPIRE k8s_psat builds on.

### Pod spec

```yaml
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: payments-svc
  containers:
    - name: app
      image: acme/payments:1.4
      volumeMounts:
        - mountPath: /var/run/secrets/aws
          name: aws-token
  volumes:
    - name: aws-token
      projected:
        sources:
          - serviceAccountToken:
              audience: sts.amazonaws.com
              expirationSeconds: 3600
              path: token
```

### The resulting JWT

```json
{
  "iss": "https://oidc.eks.eu-west-1.amazonaws.com/id/ABC123",
  "sub": "system:serviceaccount:billing:payments-svc",
  "aud": "sts.amazonaws.com",
  "exp": 1800003600,
  "iat": 1800000000,
  "kubernetes.io": {
    "namespace": "billing",
    "serviceaccount": { "name": "payments-svc", "uid": "…" },
    "pod":            { "name": "payments-svc-7df4f", "uid": "…" }
  }
}
```

### Discovery — the cluster as an OIDC issuer

```
$ kubectl get --raw /.well-known/openid-configuration
{ "issuer": "https://oidc.eks.eu-west-1.amazonaws.com/id/ABC123",
  "jwks_uri": "https://oidc…/keys",
  "response_types_supported": ["id_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"] }
```

### Trust policy hygiene

- Pin `sub` in the IAM role's trust policy — not just `iss` + `aud`.
- Without a sub-pin, every SA in the cluster can assume the role.
- For cross-account / cross-cluster, pin the cluster's OIDC issuer URL exactly.

---

## Slide 14 — Service Mesh + JWT — End-User Identity at the Sidecar

mTLS gives you *workload* identity. JWT validation at the sidecar gives you *end-user* identity. Together you get authz that knows both who is calling and on whose behalf.

### Istio RequestAuthentication + AuthorizationPolicy

```yaml
# 1. Validate JWTs from this issuer
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata: { name: jwt-acme, namespace: billing }
spec:
  selector: { matchLabels: { app: ledger-svc } }
  jwtRules:
    - issuer: "https://login.acme.com/"
      jwksUri: "https://login.acme.com/.well-known/jwks.json"
      audiences: ["https://api.acme.com/ledger"]

---
# 2. Require a valid JWT with the right scope
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata: { name: ledger-write, namespace: billing }
spec:
  selector: { matchLabels: { app: ledger-svc } }
  action: ALLOW
  rules:
    - to: [{ operation: { methods: ["POST"], paths: ["/v1/postings"] } }]
      when:
        - key: request.auth.claims[scope]
          values: ["postings:write"]
        - key: request.auth.claims[iss]
          values: ["https://login.acme.com/"]
```

### Two audiences, two enforcement layers

- **mTLS** (workload-to-workload) — enforces "*caller is the payments service in our mesh*".
- **JWT** (user-to-service, propagated through) — enforces "*and the user behind that call has consented to write postings*".
- Both must pass.

### JWT propagation between services

- Edge gateway validates the user's token, may downscope or exchange it.
- Internal services accept the same JWT in `Authorization` or in a custom header.
- For multi-hop calls preserving end-user identity: Token Exchange (RFC 8693) at the boundary.

### Don't trust the JWT silently

If `RequestAuthentication` isn't present, Istio doesn't validate the JWT — but `AuthorizationPolicy` may still *read claims from it*. The result: an attacker forges any JWT they like and your policy reads the unsigned claims. Always pair the two.

---

## Slide 15 — Workload Identity for AI Agents & MCP Servers

### Why agents make this harder

- An agent calls many MCP servers, often in different trust domains.
- The agent itself is a workload — it needs an identity.
- The MCP server's authz often wants to distinguish *the human user* from *the agent acting on their behalf*.
- Tokens flow through more hops than typical SaaS — agent → gateway → MCP server → upstream API.

### A defensible pattern

1. Agent runs in a pod with a SPIFFE ID — that identity proves the binary & tenant.
2. Human user authenticates separately via OAuth/OIDC; the agent holds a refresh token.
3. Agent → MCP Gateway over mTLS using SPIFFE; user's access token in `Authorization`.
4. MCP Gateway validates both: workload identity (mesh) *and* user identity (JWT validation).
5. Downstream MCP server uses Token Exchange to obtain a token *"user U via agent A"* for upstream APIs.

### "Agent identity" is its own claim

- Some emerging conventions: include an `agent` claim in the access token (or in the act-on-behalf-of structure of a token-exchanged JWT).
- MCP server can then audit: *"action X taken by user U via agent A on date D"*.
- Authorisation policies can demand human-confirmed actions for high-risk operations even when the agent has the right scope.

### Cross-references

This is where **OAuth_for_MCP** + **Advanced_OpenID_Connect** (Token Exchange + workload OIDC) + this deck (workload identity + service-mesh enforcement) meet.

---

## Slide 16 — Choosing a Stack — Mesh vs SPIRE vs Cloud-Native

| You're starting with... | Best fit | Why |
|---|---|---|
| One Kubernetes cluster, one cloud, low-friction mTLS | **Linkerd** or **Cilium service-mesh mode** | Identity + mTLS work out of the box; no extra system to operate. |
| Many K8s clusters, many clouds, polyglot workloads | **SPIRE** + a thin mesh | Vendor-neutral SPIFFE identity travels everywhere. |
| Heavily AWS-centric, EKS + Lambda + EC2 | **IRSA** + AWS App Mesh / Cloud Map / native VPC controls | Cloud-managed; deep IAM integration. |
| Need fine L7 traffic policy (canary, circuit breaking, custom Envoy filters) | **Istio** | Widest L7 surface; ext_authz for complex rules. |
| "We just want pod-to-pod NetworkPolicy with identity" | **Cilium** | eBPF in the kernel; no sidecars; identity-aware NetworkPolicy. |
| Compliance needs hardware-rooted attestation | SPIRE with TPM / AWS Nitro / Azure Confidential / GCP Shielded VM plugins | Workload attestation rooted in hardware. |
| Multi-org B2B service-to-service | SPIRE federation | Trust-bundle exchange enables verifiable cross-org calls without VPNs. |

### An honest trade-off

Cloud-native is the lowest-cost path inside one cloud — but you re-build it each time you add a cloud. SPIRE costs more to operate up front, pays back with portability. Mesh-based identity gets you authz *between* services for free; less useful for "identity for AWS API calls".

---

## Slide 17 — Migration Patterns

### Static keys → IRSA / WI

1. Inventory every long-lived AWS / GCP / Azure key in app code, config, secrets.
2. Create a per-workload IAM role / GCP SA / Entra app with the same permissions.
3. Configure the cluster's OIDC issuer in the cloud's STS (one-off per cluster).
4. Annotate the K8s SA and add the projected token volume.
5. Cut over one workload at a time; verify CloudTrail / Audit Logs show the new role.
6. Rotate the static key one last time, verify nothing breaks, then delete it.

### "Pinging is mTLS" → service mesh

1. Install the mesh in a single namespace, mode = permissive (mTLS optional, ALLOW unsigned).
2. Enable telemetry — see what's actually talking to what.
3. Add namespace-wide DENY policy in *audit* mode; review denies for false positives.
4. Move to mTLS=STRICT once telemetry is clean.
5. Move DENY policies to enforcing mode.

### Single cluster → SPIRE federation

1. Stand up one SPIRE Server per trust domain.
2. Issue SVIDs to a small canary set first.
3. Federate trust bundles between trust domains via the SPIFFE Federation API.
4. Verify cross-domain mTLS handshakes succeed.
5. Roll out to remaining workloads behind a feature flag.

### Lessons from real migrations

- Always start in audit / permissive mode.
- Telemetry first, policy second.
- Don't migrate *and* introduce a mesh in the same quarter — pick one source of complexity.
- The legacy permissive mode often quietly becomes permanent. Build a deadline + dashboard to prevent this.

---

## Slide 18 — Production Gotchas

- **"My mesh policy isn't enforcing"** — Workload uses a port the mesh isn't proxying (headless service, host-network pod). Confirm the proxy is in the path.
- **SVIDs that keep expiring under load** — Workload not refreshing the cert before expiry; SPIFFE Workload API call rate-limited or blocked. Re-fetch ahead of expiry (~ 50 % of TTL), with retries.
- **Trust bundle drift** — One cluster has the new CA, another still has the old. Calls fail mTLS validation.
- **IRSA: "the role is right, but it doesn't work"** — Almost always: the SA annotation is wrong, the IAM role's trust policy doesn't pin the right SA, or the cluster's OIDC issuer isn't registered as an STS identity provider.
- **Linkerd / Istio JWT skew** — Sidecar's view of "now" diverges from the IdP's. NTP everywhere, leeway ≤ 60 s.
- **"Mesh is up but my pod can't reach the internet"** — mTLS strict mode + outbound to a non-mesh endpoint = handshake failure. Configure egress policy explicitly.
- **Cilium L7 policies on TLS-encrypted traffic** — You can't inspect HTTP that you can't decrypt. Either terminate TLS at the proxy or accept L4-only enforcement.
- **Forgotten attestation selectors** — Loose → wrong identity issued to the wrong pod. Tight → pod won't get an identity after a routine deploy. Pin namespace + SA + image registry.

---

## Slide 19 — Summary & References

### What we covered

- Workload identity as a problem
- SPIFFE specification — IDs, SVIDs, federation
- SPIRE — server / agent, attestation, registration entries
- Service mesh AuthZ — what the sidecar gives you
- Istio AuthorizationPolicy patterns and pitfalls
- Linkerd's three-CRD model
- Cilium L4/L7 + Tetragon for runtime enforcement
- mTLS at scale — issuance, rotation, observability, root rotation
- Cloud-native workload identity (IRSA / WI / Managed Identity / WIF)
- K8s ServiceAccountTokenVolumeProjection
- Service mesh + JWT for end-user identity at L7
- Workload identity for AI agents & MCP servers
- Choosing a stack · migration · production gotchas

### Three take-aways

1. **Identity comes from the substrate.** The workload doesn't carry a secret; the platform attests what it is.
2. **mTLS is the easy part; rotation is the hard part.** If you can't rotate the CA root with one Slack message, you're not done.
3. **Workload identity + end-user identity must both be enforced.** mTLS alone says "this binary"; JWT alone says "this user". Real authz needs both.

### One-line takeaway

Replace every long-lived service credential with a short-lived, attested, automatically-rotated identity. The platforms exist; the only thing left is to wire them up.

### References

SPIFFE specification (spiffe.io) · SPIRE — spiffe.io/docs · *"Solving the Bottom Turtle"* (SPIFFE/SPIRE book) · Istio Security · Linkerd Authorization Policy · Cilium Network Policies · cert-manager + Trust Manager · AWS IRSA / EKS Pod Identity · GCP Workload Identity / WIF · Azure Workload Identity · NIST SP 800-204A (Service Mesh) · CNCF TAG-Security: Workload Identity
