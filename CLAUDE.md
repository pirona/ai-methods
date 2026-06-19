# CLAUDE.md — Claude Code Operator Template

> **Public template for Claude Code configuration.**
> This file captures a complete operator profile for a Linux infrastructure engineer
> with a self-hosted homelab. Fork it, adapt the sections to your own stack,
> and replace `[FILL IN: ...]` markers with your own values.
> Sections on TLS rules, CI conventions, and general guidelines are universal
> and can be used as-is or lightly adjusted.
>
> Original profile: Linux/K8s infrastructure, self-hosted services, HAProxy TLS termination,
> GitOps with ArgoCD, electronic music production as a side domain.

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

#### Push to self-hosted Git — always via SSH

Non-standard SSH port common on self-hosted instances — always check. URL format:

```
ssh://git@<your-git-host>:<port>/<username>/<repo>.git
```

Configure the remote:

```bash
git remote set-url origin ssh://git@<your-git-host>:<port>/<username>/<repo>.git
```

SSH keys on the workstation must be pre-authorized on the instance.
**Never use HTTPS for git push to a self-hosted instance**: CLI HTTPS auth is often not configured.

#### Self-hosted Git API (REST)

Store the API token in a local file outside the repository.
Never embed the token value in a URL or shell argument.
Pass it only via header:

```bash
curl -H "Authorization: Bearer $(cat /path/to/.token | tr -d '\n')" \
     https://<your-git-host>/api/v1/...
```

---

## 6. CI methodology: self-hosted Git (no runner) → push mirror → GitHub Actions

> Reusable template for any project hosted on a self-hosted Gitea instance
> (no local CI runner), with automated builds via free GitHub Actions runners.

### Architecture

```
Local development
       ↓  git push / git push origin <tag>
  Self-hosted Gitea (private)   ← source of truth, no runner
       ↓  automatic push mirror
  GitHub (public or private)    ← passive mirror
       ↓  tag v*.*.*
  GitHub Actions                ← CI/CD, build, release
```

### Configuring the push mirror in Gitea

Gitea → Settings → Repository → **Push Mirrors**:
- Mirror URL: `https://github.com/USER/REPO.git`
- Credentials: GitHub token with `repo` scope
- Sync: immediate on every push

All commits, branches, and tags pushed to Gitea are automatically replicated to GitHub.
Never push directly to GitHub — the self-hosted instance is the source of truth.

### Reference GitHub Actions workflow

```yaml
name: Release

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  build:
    # Guard clause: only run on GitHub, not on Gitea
    # (safety net if self-hosted Gitea gains a runner later)
    if: github.server_url == 'https://github.com'
    runs-on: ubuntu-latest
    permissions:
      contents: write   # required to create a GitHub Release

    steps:
      - uses: actions/checkout@v4
      # … project-specific build steps …
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          files: dist/*
```

**Key points:**
- Trigger `push.tags: ['v*.*.*']` — intentional builds, not on every commit.
- Guard `if: github.server_url == 'https://github.com'` — safe if architecture evolves.
- `permissions: contents: write` — required for `action-gh-release`.
- Secrets (keystores, tokens): stored in GitHub → Settings → Secrets, never in the repo.

### Automated changelog generation

Extracts `feat:` and `fix:` commits between the previous tag and HEAD:

```bash
PREV_TAG=$(git tag --sort=-version:refname | grep -v "^${{ github.ref_name }}$" | head -1)

FEATS=$(git log "${PREV_TAG}..HEAD" --pretty=format:"%s" --no-merges \
  | grep -E "^feat(\([^)]+\))?: " \
  | sed -E 's/^feat(\([^)]+\))?: //' \
  | sed 's/^/- /')

FIXES=$(git log "${PREV_TAG}..HEAD" --pretty=format:"%s" --no-merges \
  | grep -E "^fix(\([^)]+\))?: " \
  | sed -E 's/^fix(\([^)]+\))?: //' \
  | sed 's/^/- /')
```

Requires commit messages following [Conventional Commits](https://www.conventionalcommits.org/).

### Release workflow

```bash
# Normal development
git commit -m "feat(scope): description"
git push                        # mirrored to GitHub automatically

# Trigger a release
git tag v1.x.y
git push origin v1.x.y          # tag mirrored → Actions triggered → Release created
```

---

## 7. Automation platform — Known constraints

> Applies to n8n or any similar workflow automation platform.

- Use persistent workflow state (e.g. n8n's `getWorkflowStaticData('global')`) for
  cross-execution deduplication
- Some downstream services reject SVG URLs — use direct JPG URLs instead
- Dynamic array expressions can be problematic with certain destinations:
  prefer hardcoded item objects when needed
- Avoid OAuth2 race conditions on shared credentials across workflows:
  stagger schedules, add retry logic

---

## 8. Kubernetes (K3s)

- Cluster: 1 control plane + 2 workers
- GitOps via **ArgoCD**
- Patch management: ansible-driven, mandatory drain/uncordon procedure
- Ingress: managed by external HAProxy (no ingress-nginx for public exposure)
- Secrets: never commit secrets in plaintext — use sealed-secrets or Vault

---

## 9. Domains & DNS

- All public traffic goes through HAProxy (see section 4)
- Existing subdomains must be checked before any modification
- For internal environment examples: use `service-a.example.internal`,
  `service-b.example.internal` as generic placeholders

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

---

## 11. Security by design

Security hardening is a first-class design constraint from day 1 of every project —
not a post-deployment audit or an optional hardening pass.

- **Least privilege by default**: users, services, and network rules get only the minimum
  permissions they need to function. Never start permissive and tighten later.
- **No open surfaces without justification**: no `0.0.0.0` binds, no open ports,
  no world-readable files without a documented reason.
- **Secrets management defined before the first commit**: Ansible Vault, sealed-secrets,
  GitHub Secrets, or equivalent — never plaintext secrets in the repository.
- **TLS everywhere feasible**: even on internal services. Use the internal CA correctly
  (see section 4) rather than disabling verification.
- **Audit logging in the architecture phase**: plan what needs to be logged and where
  before writing the first line of code — retrofitting observability is expensive.
- **Dependency hygiene**: pin versions, verify checksums, prefer minimal base images.
  Every external dependency is an attack surface.
- **Threat model first**: before proposing an architecture, identify the trust boundaries
  and what an attacker could target. Design to minimize blast radius.

---

## 12. Documentation philosophy

Documentation is written from the first day of a project and evolves alongside the work
until completion. It is never a post-mortem dump or an afterthought.

- **Simple**: clear language, no jargon without definition, structure that a newcomer
  can follow without prior context.
- **Exhaustive**: covers prerequisites, context, architecture decisions, operational
  procedures, and known limitations.
- **Pedagogical**: explains *what* we do AND *why* — not just a sequence of commands
  to copy-paste. A reader should understand the reasoning, not just be able to repeat
  the steps blindly.
- Every command block is accompanied by an explanation of what it does and what
  to verify afterward (expected output, side effects, rollback procedure).
- Architecture decisions — why this tool and not that one — are documented at the time
  they are made, not reconstructed from memory six months later.
- Documentation lives in the repository alongside the code it describes.
  If the code changes, the documentation changes in the same commit.
