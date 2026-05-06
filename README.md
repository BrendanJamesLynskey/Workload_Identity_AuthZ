# 🛡 Workload Identity & Service-Mesh AuthZ

An interactive Reveal.js presentation on **service-to-service authorisation** — how workloads prove who they are without shared secrets, how the service mesh enforces policy on top of those identities, and how cloud-native federation (IRSA / Workload Identity / WIF) ties it back to cloud APIs.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/Workload_Identity_AuthZ/)

## 📄 [Markdown Version](presentation.md)

## 📚 Companion decks — [Authorization Models](https://brendanjameslynskey.github.io/Authorization_Models/) · [Edge & Gateway AuthZ](https://brendanjameslynskey.github.io/Edge_and_Gateway_AuthZ/) · [Advanced OpenID Connect](https://brendanjameslynskey.github.io/Advanced_OpenID_Connect/)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Title | Attest → Issue → Verify → Authorise |
| 02 | Topics | The four chapters mapped out |
| 03 | The Workload-Identity Problem | Why user-AuthN doesn't translate; long-lived-key crisis |
| 04 | SPIFFE — The Specification | SPIFFE ID; X.509 SVID and JWT SVID anatomy |
| 05 | SPIRE — The Reference Implementation | Server / agent / Workload API; node + workload attestation |
| 06 | Workload Attestation | Node + workload plugins; selectors; the registration entry |
| 07 | SPIFFE Federation | Trust bundles, the Federation API, multi-cluster / multi-cloud / B2B |
| 08 | Service Mesh Primer | What the sidecar gives you for AuthZ |
| 09 | Istio AuthorizationPolicy | Patterns and the four pitfalls |
| 10 | Linkerd | Server / AuthorizationPolicy / MeshTLSAuthentication CRDs |
| 11 | Cilium + Tetragon | eBPF-native L4/L7; runtime enforcement |
| 12 | mTLS at Scale | Issuance, rotation, trust-bundle distribution, root rotation |
| 13 | Cloud-Native Workload Identity | IRSA, GCP WI, Azure WI, Workload Identity Federation |
| 14 | K8s ServiceAccountTokenVolume | The substrate everything cloud-native builds on |
| 15 | Service Mesh + JWT | End-user identity at L7 alongside workload mTLS |
| 16 | Workload Identity for Agents & MCP | Agent identity + user identity composed |
| 17 | Choosing a Stack | Mesh vs SPIRE vs cloud-native — decision matrix |
| 18 | Migration Patterns | Static keys → IRSA; "ping is mTLS" → mesh; single cluster → SPIRE federation |
| 19 | Production Gotchas | The most common things that go wrong |
| 20 | Summary | Three take-aways and references |

---

## Audience

- Engineers shipping services into Kubernetes / multi-cloud who want to retire long-lived API keys.
- Architects choosing between SPIRE, a service mesh, and cloud-native workload identity.
- Platform teams operating mTLS at scale (cert rotation, trust-bundle distribution).
- Anyone wiring an AI agent into an MCP-style stack and asking "what's the agent's identity?"

This deck assumes the OAuth/OIDC foundations from the Identity & Access series; it focuses on *workload* identity rather than *user* identity.

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono · inline SVG diagrams.

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## See also

- [Authorization Models](https://github.com/BrendanJamesLynskey/Authorization_Models) — the conceptual underpinning (RBAC/ABAC/ReBAC, OPA/Cedar).
- [Edge & Gateway AuthZ](https://github.com/BrendanJamesLynskey/Edge_and_Gateway_AuthZ) — the north-south complement to this deck's east-west focus.
- [Advanced OpenID Connect](https://github.com/BrendanJamesLynskey/Advanced_OpenID_Connect) — workload OIDC slide deepens here.
- [OAuth for MCP Servers](https://github.com/BrendanJamesLynskey/OAuth_for_MCP) — the Docker MCP Gateway pattern.
- [Cloud_aaS_05_Cloud_Security](https://github.com/BrendanJamesLynskey/Cloud_aaS_05_Cloud_Security) — the wider cloud-security context.

## References

SPIFFE specification (spiffe.io) · SPIRE — spiffe.io/docs · *"Solving the Bottom Turtle"* (SPIFFE/SPIRE book, Sullivan et al.) · Istio Security · Linkerd Authorization Policy docs · Cilium Network Policies · cert-manager + Trust Manager · AWS IRSA / EKS Pod Identity · GCP Workload Identity / Workload Identity Federation · Azure Workload Identity · NIST SP 800-204A (Service Mesh) · CNCF TAG-Security: Workload Identity in Multi-System Environments

## License

Educational use. Code examples provided as-is.
