# CLAUDE.md — Global Instructions

> Global instructions for Claude Code. This file describes the full technical environment,
> working conventions, and hard rules that apply across all projects.

---

## 1. Identity & operator context

- **Role:** Linux Systems & Network Engineer — CKA certified, infrastructure specialist
- **Creative domain:** Electronic music composer / visual artist (trip-hop/downtempo)
- **Communication:** Direct, concise, technical. No unnecessary preamble.
  Undiagnosed neurodivergence — prefer structured responses, no ambiguity,
  with a clear information hierarchy.
- **Language:** French by default for conversation, English for code and config files.

---

## 2. Homelab infrastructure

### Hypervisor & compute

| Component       | Detail                                        |
|-----------------|-----------------------------------------------|
| Hypervisor      | Proxmox VE on low-power x86 SBC               |
| K8s cluster     | K3s — 1 control plane, 2 workers              |
| NAS             | OpenMediaVault                                |
| Storage         | ZFS mirror pool + hardware RAID (parallel)    |
| Monitoring      | Lightweight push-based monitoring stack       |

### Network

- **Router:** Prosumer router with VLAN support
- **Segmentation:** VLANs (isolate critical services from client LANs)
- **Front reverse proxy:** HAProxy on a hardened dedicated VM (see section 4)

### Active self-hosted services

| Service              | Notes                                       |
|----------------------|---------------------------------------------|
| Automation platform  | Workflow automation (n8n-style)             |
| Self-hosted Git      | Internal SCM                                |
| Music streaming      | Pinned to a specific stable version         |
| Wiki platform        | Personal knowledge base                     |
| Social scheduler     | Social media post scheduling                |
| Analytics platform   | Self-hosted web analytics                   |
| Blog platform        | Multiple blogs on dedicated LXC containers  |
| File sync platform   | Avoid special characters in base64 secrets  |

---

## 3. Reference tech stack

### IaC & orchestration

```
Kubernetes (K3s / CKA)   Ansible / AWX
Terraform                ArgoCD
GitLab CI/CD             Proxmox API
```

### Linux & security

- Target OS: **Debian / Ubuntu LTS** (servers), **Ubuntu Studio** (music workstation)
- Hardening: **ANSSI** Linux hardening guidelines
- Internal CA: certificates go in `/usr/local/share/ca-certificates/`,
  then `update-ca-certificates` — never disable TLS verification
- ZFS: ARC tuning, snapshots via **sanoid**, always add disks in pairs

### Main workstation

```
CPU   Modern Intel high-core-count desktop
RAM   32 GB DDR5
OS    Windows 11
WSL   Present but limited use
```

---

## 4. HAProxy & TLS certificates — Hard rules

> ⚠️ This section is critical. Every TLS configuration suggestion must follow these rules.

### Architecture

```
Internet
   │
   ▼
HAProxy (hardened VM)
   │  ─── TLS termination here ───
   │  Let's Encrypt via certbot/acme
   ▼
Internal services (plain HTTP or mTLS depending on service)
```

### Let's Encrypt constraints

- **No wildcards**: each subdomain gets its own dedicated certificate.
  Example: `service-a.example.com`, `service-b.example.com` → 2 separate certs.
- When adding a new service, **always provision**:
  1. A Let's Encrypt cert for the exact FQDN (HTTP-01 or DNS-01 challenge)
  2. The corresponding HAProxy frontend
  3. Bind `ssl crt /etc/haproxy/certs/<fqdn>.pem`
- Never suggest a wildcard cert — it is not in use.
- Renewal is per-cert: automate certbot with HAProxy reload hooks.

### HAProxy frontend template (reference)

```haproxy
frontend https_<service>
    bind *:443 ssl crt /etc/haproxy/certs/<fqdn>.pem
    http-request set-header X-Forwarded-Proto https
    default_backend bk_<service>

backend bk_<service>
    server <service>_1 <internal_ip>:<port> check
```

---

## 5. Code & CI/CD conventions

### Ansible

- Lint: **ansible-lint** + **PEP8** on all playbooks
- No hardcoded secrets — use Ansible Vault or environment variables
- Mandatory idempotence: every task must be safely re-runnable

### GitLab CI/CD

- Pipelines defined in versioned `.gitlab-ci.yml`
- Sensitive variables in CI/CD Variables (masked + protected)
- Canonical stages: `validate → build → test → deploy`

### Terraform

- Remote state backend (never local in prod)
- `terraform fmt` + `terraform validate` before any commit
- Modules explicitly versioned

### Git (self-hosted)

- Convention: imperative present tense, English (`Add`, `Fix`, `Remove`)
- No direct commits to `main` — MR/PR mandatory

---

## 6. Automation platform — Known constraints

- `getWorkflowStaticData('global')` for cross-execution deduplication
- Some downstream services reject SVG URLs — use direct JPG URLs instead
- Dynamic array expressions can be problematic with certain destinations:
  prefer hardcoded item objects when needed
- Avoid OAuth2 race conditions on shared credentials across workflows:
  stagger schedules, add retry logic

---

## 7. Kubernetes (K3s)

- Cluster: 1 control plane + 2 workers
- GitOps via **ArgoCD**
- Patch management: ansible-driven, mandatory drain/uncordon procedure
- Ingress: managed by external HAProxy (no ingress-nginx for public exposure)
- Secrets: never commit secrets in plaintext — use sealed-secrets or Vault

---

## 8. Domains & DNS

- All public traffic goes through HAProxy (see section 4)
- Existing subdomains must be checked before any modification
- For internal environment examples: use `service-a.example.internal`,
  `service-b.example.internal` as generic placeholders

---

## 9. Audio / Music (workstation)

- **Current DAW:** Ableton Live (Windows)
- **Target Linux DAW:** Bitwig Studio (pending Windows stabilization)
- **Plugins:** Mastering suite (iZotope-style), mix assistant, limiter (FabFilter-style)
- **Future Linux audio stack:** JACK / PipeWire + Windows VST bridge
- Do not suggest ASIO approaches in a Linux context

---

## 10. General rules for Claude Code

1. **Always verify** that a dedicated Let's Encrypt cert exists before proposing
   a public URL for a new service.
2. **Never disable** TLS verification, even in dev — use the internal CA correctly.
3. **Prefer** low-maintenance-overhead solutions (no exotic dependencies).
4. **Alerts over active monitoring** — monitoring configs should favor push alerts
   (webhooks, notification services) over manual polling.
5. **No bloat** — if a dependency is not strictly necessary, don't add it.
6. **Prefer European solutions** when the choice is neutral (data sovereignty).
7. When multiple approaches are valid, **list options with trade-offs**
   rather than imposing a choice.
8. Shell commands must be **directly copy-pasteable** — no ambiguous placeholders
   without explicit explanation of what to substitute.
9. **ZFS:** always add disks in pairs, never break a mirror to save cost.
10. Every new homelab service requires: dedicated cert → HAProxy frontend → backend → test.
11. **Documentation language:** Public-facing documentation must be written in **English**.
    Private documentation (personal wiki) in **French**.
    If the destination is not specified, **always ask** before writing.
12. **App licensing:** All produced or published applications must be licensed under
    **GNU GPL v3**. Include a `LICENSE` file (full GPL v3 text) and the SPDX identifier
    `SPDX-License-Identifier: GPL-3.0-or-later` at the top of source files.
13. **Information sources:** When searching for anything (commands, configs, APIs, best practices),
    only consult **official documentation** from the relevant project, vendor, or standards body.
    Forbidden sources: Stack Overflow, Reddit, third-party forums, personal blogs,
    unofficial community wikis. When in doubt about a source's legitimacy, don't use it.
