# CLAUDE.md — Claude Code Operator Template

> **Public template for Claude Code configuration.**
> This file captures a complete operator profile for a Linux infrastructure engineer
> with a self-hosted homelab. Fork it, adapt the sections to your own stack,
> and replace the generic descriptions and `<placeholders>` with your own values.
> Sections on TLS rules, CI conventions, and general guidelines are universal
> and can be used as-is or lightly adjusted.
>
> Original profile: Linux/K8s infrastructure, self-hosted services, HAProxy TLS termination,
> GitOps with ArgoCD, electronic music production as a side domain.
>
> See `APPENDIX-skills.md` for the shared-skills pattern (reusable procedures loaded on demand).

---

## 1. Identity & operator context

- **Role:** Linux Systems & Network Engineer — CKA certified, infrastructure specialist
- **Creative domain:** Electronic music composer / visual artist (trip-hop/downtempo)
- **Communication:** Direct, concise, technical. No unnecessary preamble.
  Prefer structured responses, no ambiguity, with a clear information hierarchy.
  Often gives very short instructions ("restart the build", "commit & push everything") —
  no detailed explanations needed unless something goes wrong.
- **Language:** French by default for conversation, English for code and config files.

---

## 2. Homelab infrastructure

### Hypervisor & compute

| Component       | Detail                                        |
|-----------------|-----------------------------------------------|
| Hypervisor      | Proxmox VE on low-power x86 SBC               |
| K8s cluster     | K3s — 1 control plane, 2 workers              |
| NAS             | OpenMediaVault                                |
| Storage         | Directory storage on the hypervisor (qcow2 VMs, raw containers — no snapshot for containers); NAS disks without redundancy |
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
| Push notifications   | Auth on, default deny; one restricted write-only account per integration, one topic each |

---

## 3. Reference tech stack

### IaC & orchestration

```
Kubernetes (K3s / CKA)   Ansible
ArgoCD                   GitLab CI/CD
Proxmox API
```

### Linux & security

- Target OS: **Debian / Ubuntu LTS** (servers), **Ubuntu Studio** (music workstation)
- Hardening: **ANSSI** Linux hardening guidelines
- Internal CA: certificates go in `/usr/local/share/ca-certificates/`,
  then `update-ca-certificates` — never disable TLS verification

### Main workstation

```
CPU   Modern Intel high-core-count desktop
RAM   32 GB DDR5
OS    Windows 11
WSL   Actively used (file access to the other drives, interop with Windows processes)
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

### Code comments

- **Synthetic and concise**: a comment explains the *why*, never the *what* — the code itself expresses what.
- **Language**: match the language already in use in the repository. Do not comment in French if the rest of the code is in English, and vice versa.
- No multi-line blocks, no essay docstrings — one short line when strictly necessary.

### Ansible

- Lint: **ansible-lint** + **PEP8** on all playbooks
- No hardcoded secrets — use Ansible Vault or environment variables
- Mandatory idempotence: every task must be safely re-runnable

### GitLab CI/CD

- Pipelines defined in versioned `.gitlab-ci.yml`
- Sensitive variables in CI/CD Variables (masked + protected)
- Canonical stages: `validate → build → test → deploy`

### Git (self-hosted)

- Convention: imperative present tense, English (`Add`, `Fix`, `Remove`)
- No direct commits to `main` — MR/PR mandatory
- **Always `git pull`/`git fetch` before any `git push`**, in every repo — don't wait for
  a rejected push to discover the remote has diverged.

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

## 6. Credentials & token management

Store tokens in files outside any repository, never in URLs or shell arguments.

```
~/.tokens/<service>_pat   # chmod 600, parent dir chmod 700
```

The token location can differ per workstation: write it once per machine in the config, and never assume it.

For GitHub HTTPS operations, register via git credential store:

```bash
mkdir -p ~/.tokens && chmod 700 ~/.tokens
printf '<token>' > ~/.tokens/github_pat && chmod 600 ~/.tokens/github_pat
printf "https://<username>:%s@github.com\n" "$(cat ~/.tokens/github_pat)" >> ~/.git-credentials
git config --global credential.helper store
```

Global gitignore (`~/.config/git/ignore`) must include token filename patterns:

```
.token
.token*
*.token
github_pat
github_pat*
```

---

## 7. Wiki — Multi-platform workflow (self-hosted Git + GitHub)

> Applies to any project hosted on a self-hosted Git instance with a push mirror to GitHub.

Wiki repos are **independent git repositories**, separate from the code repo.
Push mirrors (code repo) do **not** cover the wiki repos.
Both wikis must be pushed independently on every documentation update.

### Repository URLs

```
Self-hosted wiki  :  ssh://git@<your-git-host>:<port>/<username>/<repo>.wiki.git  (branch: <default-branch>, see below)
GitHub wiki       :  https://github.com/<github-user>/<repo>.wiki.git             (branch: master)
```

**Never hard-code the self-hosted wiki branch.** The Gitea wiki UI reads only the repository's default
branch (HEAD), whatever it is — it depends on the instance's `DEFAULT_BRANCH` when the wiki repo was
created, and it can change over time. Pushing to another branch silently creates an orphan branch that
the UI never shows. Always check before pushing:

```bash
git ls-remote --symref ssh://git@<your-git-host>:<port>/<username>/<repo>.wiki.git HEAD
# → ref: refs/heads/<default-branch>
```

GitHub wikis, on the other hand, stay on `master`, even when the code repo moved to `main`.

### Initial setup

```bash
# Clone from self-hosted Git
git clone ssh://git@<your-git-host>:<port>/<username>/<repo>.wiki.git wiki-<repo>/
cd wiki-<repo>/

# Add GitHub remote (credential store provides the token automatically)
git remote add github https://github.com/<github-user>/<repo>.wiki.git

# Push both
git push origin <default-branch>
git push github master --force   # --force only if GitHub had an init placeholder page
```

> **GitHub prerequisite:** the GitHub wiki repo does not exist until a first page is created
> via the UI (`github.com/<user>/<repo>/wiki` → "Create the first page"). Do this before the first push.

### Ongoing updates

```bash
cd wiki-<repo>/
# … edit .md files …
git add -A && git commit -m "docs: <description>"
git push origin <default-branch>   # self-hosted Git
git push github master   # GitHub
```

### README links

Use **dual links** — each platform points to its own wiki:

```markdown
Documentation: [Gitea](https://<your-git-host>/<username>/<repo>/wiki) · [GitHub](https://github.com/<github-user>/<repo>/wiki)

| Page | Self-hosted | GitHub |
|------|-------------|--------|
| Architecture | [↗](https://<your-git-host>/<username>/<repo>/wiki/Architecture) | [↗](https://github.com/<github-user>/<repo>/wiki/Architecture) |
```

Never use a single link that only resolves correctly on one platform.

---

## 8. LLM Council — Multi-model strategic decisions

> Credit: [LLM Council skill](https://github.com/aiwithremy/claude-skills-llm-council)
> by [@aiwithremy](https://github.com/aiwithremy) — genuinely excellent, highly recommended.

For high-stakes design decisions and project conception, the LLM Council skill is outstanding.
It convenes multiple frontier models simultaneously (Claude, GPT-4o, Gemini, etc.) and
synthesizes their perspectives into a single, consolidated answer.

**Why it matters:** a single model has blind spots. Architectural decisions made with only
one model's judgment can silently inherit its biases or miss entire categories of risk.
The Council surfaces disagreements between models, which is exactly where the interesting
trade-offs live.

**When to use it:**
- Architecture choices with long-term consequences (storage backend, auth strategy, infra topology)
- Technology selection where trade-offs are non-obvious
- Security model design before the first line of code
- Any decision where "getting it wrong" would be expensive to reverse

**Setup note:** the skill triggers on English prompts by default. Configure the response
language to match your working language if needed.

Use via Claude Desktop, not Claude Code — the skill runs as a slash command in the Desktop app.

---

## 9. CI methodology: self-hosted Git (no runner) → push mirror → GitHub Actions

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

### Build sequencing — hard rule

**One build at a time.** Never push a tag (or any other CI trigger) while a previous
build is still running.

- After each `git push origin vX.Y.Z`: wait for explicit confirmation that the build
  passed before tagging again.
- If multiple commits are ready: push them to `main` without a tag, then tag only
  when the time comes.
- Never batch-push tags that each trigger independent CI builds.

Rationale: concurrent builds waste runner minutes, produce race conditions on release
asset uploads, and make it harder to bisect which tag introduced a regression.

---

## 10. Automation platform — Known constraints

> Applies to n8n or any similar workflow automation platform.

- Use persistent workflow state (e.g. n8n's `getWorkflowStaticData('global')`) for
  cross-execution deduplication
- Some downstream services reject SVG URLs — use direct JPG URLs instead
- Dynamic array expressions can be problematic with certain destinations:
  prefer hardcoded item objects when needed
- Avoid OAuth2 race conditions on shared credentials across workflows:
  stagger schedules, add retry logic

---

## 11. Kubernetes (K3s)

- Cluster: 1 control plane + 2 workers
- GitOps via **ArgoCD**
- Patch management: ansible-driven, mandatory drain/uncordon procedure
- Ingress: managed by external HAProxy (no ingress-nginx for public exposure)
- Secrets: never commit secrets in plaintext — use sealed-secrets or Vault

---

## 12. Domains & DNS

- All public traffic goes through HAProxy (see section 4)
- Existing subdomains must be checked before any modification
- For internal environment examples: use `service-a.example.internal`,
  `service-b.example.internal` as generic placeholders

---

## 13. General rules for Claude Code

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
9. **NAS storage:** the disks have no redundancy — never suggest software RAID or ZFS without prior discussion.
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
14. **Sequential by default**: never run multiple actions in parallel without explicit approval.
    If parallelism seems worthwhile (truly independent actions, significant time gain), submit
    the opinion for validation before acting.
15. **Never kill, never delete without explicit approval.** No deletion (file, workflow, credential,
    cluster resource, post, snapshot, branch, API object) and no stopping of a process, pod or service
    (`kill`, `kubectl delete`, HTTP `DELETE`…) without a request or validation for that exact action —
    including test objects Claude created itself. Flag and propose, never execute on its own.
    Helper processes started for a test should stop by themselves (`timeout 150 <command>`), not be killed.
16. **Throwaway examples and tests are non-impacting**: read-only, fake or disposable data, never a real
    publication, deletion or overwrite on a real system. (Optional: slip a small joke into the sample
    content — it makes test artefacts easy to recognize and to clean up.)

---

## 14. Security by design

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

## 15. Documentation philosophy

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

---

## 16. Working Methodology

### Strategic questions / vague instructions

Use **LLM Council** (Claude Desktop skill) to get cross-model perspectives before deciding.
If unavailable: list options with trade-offs, wait for selection.

### Precise orders — mandatory sequence

1. **Review existing state** — read code, configs, and related services before any action.
2. **What do we want?** — restate the objective in one clear sentence and submit it.
3. **How will we do it?** — complete action plan submitted for validation before execution.
4. **Integration** — explicitly identify how the change integrates with the existing system
   (dependencies, interfaces, side effects).
5. **Simplicity first** — simplest, functional, maintainable solution.
   No premature abstractions, no unnecessary dependencies.

### Thinking aloud is not a request to implement

When the user describes a use case, an example, or explores an idea during a design discussion:
restate the understanding and ask explicitly "do you want me to implement this?" before touching
code. Only an explicit action request ("go", "implement", "do it") authorizes the move to code.
Lesson learned: a 150+ line feature was once coded from an example given mid-design
("my typical breakfast") and had to be reverted entirely.

### Execution rules

- **No action without prior validation.**
- **Strictly sequential**: one step at a time, in the order of the validated plan.
  No parallel or simultaneous actions without explicit agreement.

### After each implementation / deployment

Systematic code review of all interactions between what was just deployed and the existing system:
interfaces, dependencies, potential side effects.

**Feature PRs get a code review before merge** — correctness, security, interactions with what
already exists — and its conclusions are reported to the user before proposing the merge.
A pure fix is exempt.

### When the user says "ok"

1. Final code review — complete verification of all changes.
2. Project memory update — save key decisions, identified constraints, and project state.

---

## 17. Acceptance testing — real-device validation via adb

> Applies to any mobile project (Android/iOS) with access to a physical device.
> Complements section 9 (CI methodology) and section 16 (working methodology) — does not
> replace them.

A bug is only considered fixed, and a fix is only ready to be committed/tagged, after it has
been reproduced and validated on a real device — never on the strength of a code review, a
plausible hypothesis, or theoretical reasoning alone, no matter how solid it looks.

- **Device connection**: prefer adb wireless debugging (Android 11+) when the workstation
  doesn't see the phone over USB. Sequence:
  `adb pair <ip>:<pairing_port> <code>` then `adb connect <ip>:<main_port>`
  (the main port changes every time wireless debugging is toggled — ask the user again if needed).
- **Before any commit claiming to fix a bug**: reproduce the exact user scenario on the device
  (same gestures, same screen, same conditions) and confirm the buggy behavior is actually
  gone — not just that the code "looks" correct.
- **If the fix touches a server-side flow** (API save, sync): verify the result server-side
  after the action, not just the client-side display — a screen can look correct while
  persistence silently failed.
- **Never push a new build tag on the strength of an unverified hypothesis about a device
  bug.** Fix, reproduce, confirm, only then commit/tag — see section 9 (one build at a time,
  explicit user validation between tags, including when a new fix hypothesis follows a first
  attempt that failed).
- **Reproduce on test data, never on production data without explicit approval**: for a
  destructive scenario (data loss, overwrite), create/delete dedicated test records (explicit
  title pattern like `TEST_xxx_TO_DELETE`), clean up afterward.

Lesson learned: a first fix (Android keyboard resize mode) was shipped and tagged on the
strength of a plausible hypothesis, without a confirmed device repro — the CI build passed,
but the bug persisted in real usage. A second pass with an actual adb repro (real text
selection on the real screen, save, re-read the persisted content server-side) revealed the
real cause — a missing API parameter, invisible from reading the client business logic alone.

---

## 18. Local AI image generation platform — Known constraints

- Self-hosted node-based image generation tool (ComfyUI-style), reachable over HTTP on
  the local network, with workflows stored as JSON files on disk.
- A local LLM server (Ollama-style) runs on the same host as the image generation tool —
  connect to it via loopback address, not the network IP.
- Resolution guidance for modern flow-matching image models (per official model docs):
  dimensions must be multiples of 32, minimum 256px, maximum ~4MP, ~2MP recommended for
  quality/speed. Common ~1MP presets: 1024x1024 (1:1), 1152x896/896x1152 (4:3/3:4),
  1216x832/832x1216 (3:2/2:3), 1344x768/768x1344 (16:9/9:16), 1536x640/640x1536 (21:9/9:21).

### Submitting jobs

- **One image = one job**, in small quantities (6 options, not 12), fire-and-forget: one API call per
  image, results fetched afterwards from the history endpoint. Never a big batch, never a script that
  keeps the queue full.
- Never interrupt or delete a job — not even your own — without explicit approval; the user cancels
  what they want from the UI.
- If a queue-manager custom node is installed, the API call **must** carry
  `extra_data.extra_pnginfo.workflow` with an `id` and a `workflow_name`: without it the worker crashes
  and the job stays "running" forever (only a server restart clears it).
- On a RAM-limited machine: use the fp8 text encoder, never the bf16 one next to a large model, and
  free models/cache after every job. Explore with the small fast model, render the final with the large one.
- **Regenerating a figure that touches the frame**: keep the previous image, put the original render
  on a larger white canvas and pass it as a **reference image** so the drawing stays close; if the crop
  is on the sides the model tends to copy it — fall back to the original model with a larger canvas.
  The goal is 0 opaque pixels on the four borders of the final cut-out.

### Working rule for existing workflow files

- **Never edit an existing workflow file.** Always copy it to a new file with a
  `_claude.json`-style suffix (increment to `_v2` etc. if one already exists).
- To build a new workflow, find existing files that already contain the wanted patterns
  and copy their exact node structures (`type`, `inputs`, `widgets_values`) instead of
  guessing widget keys from memory.

### Known gotchas

1. **Workflow root UUID** (the top-level `"id"` field in the JSON): it identifies the
   workflow to the server. Copying a file without regenerating this UUID causes a
   `409 Conflict` on save. Always generate a fresh UUID for any copied workflow file.
2. **Duplicate custom node plugin installs**: when the same node pack is installed
   twice (e.g. two folders under the custom-node directory), schemas can drift between
   older saved workflows and what's actually loaded. **Always verify a node's real schema
   against the live server's introspection API before copying it** rather than trusting
   an example file — node names containing special characters (like `/`) can break naive
   URL-encoded lookups, so fetch the full introspection dump and filter client-side in
   that case.
3. **Save-image nodes with a "unique filename" feature**: some implementations only
   start appending a numeric suffix from the *second* saved file with a given prefix —
   the very first file gets no suffix at all. Include a dynamic placeholder (timestamp,
   seed, etc.) in the filename prefix to guarantee uniqueness from the first run.

---

## 19. Graphic design — required references

> Hard rule, across every project (image generation, artifacts, mascots/icons, layout,
> print...).

For any graphic-design request (composition, layout, shape choices, figure/ground
relationship, visual hierarchy, identity work...), always apply the operator's own reference
texts on the subject. This template ships with none by default — list your own reference
works here, e.g.:

- *(operator's reference book #1)*
- *(operator's reference book #2)*

Mobilize these principles actively in the proposal (not just cited at the end of the
response): relationship between elements and the frame, tension/rest of shapes, visual
weight of point/line/plane, economy of means. Applies as much to an art direction (e.g. a
mascot, an app's identity) as to an artifact layout or a generated poster.

### Unbridled creativity by default

For anything visual or playful (theme, illustration, animation, mascot, easter egg, layout), the
default criterion is a strong, surprising, memorable stance rather than the safe, decorative version.
A theme that is "nice but tame" is a failure. Concept first (strong metaphor, interactions, details),
then careful execution — without sacrificing legibility, contrast, or `prefers-reduced-motion`.

---

## 20. Blog / CMS posts — mandatory fields

> Applies to any post published or saved as a draft on a managed blog instance (Ghost or similar),
> whether Claude writes it from scratch or pushes/edits it through the Admin API.

A post must never stay without:

1. **A feature image** — never push or publish a post without a cover image.
2. **The meta fields filled** — `meta_title` and `meta_description`, plus the social cards
   (`og_title`/`og_description`, `twitter_title`/`twitter_description`) at least as text. The social
   *images* may stay empty (the CMS falls back to the feature image); titles and descriptions must be
   written, never left `null`.

If they were not provided, write them from the real content of the article (no generic placeholder)
before considering the task done — do not wait to be asked. Always create as **draft**; publishing or
deleting a post requires explicit approval (rule 15). Re-read the post through the API after creation
(status, meta, tags) and give the editing link.

---

## 21. Persistent memory and shared skills

> This is the architecture behind the "persistent contract" described in the README. The detail of
> how skills are built and shared is in `APPENDIX-skills.md`.

### Layers

| Layer | Content | Loaded |
|-------|---------|--------|
| Global `CLAUDE.md` | Operator profile, hard rules, one index line per project | Every session — keep it small |
| `memory/NN-slug.md` | One file per project: state, decisions, pitfalls, procedures | Only when the task touches that project |
| `memory/NN-slug-journal.md` | Chronological log of a large project (the short file keeps state + a table of contents with line numbers) | On demand, by line range or `grep -n` |
| `memory/transverse-rules.md` | Rules valid across projects (collaboration, safety, images…) | Before visual work, PRs, or any destructive action |
| `skills/<name>/SKILL.md` | Repeated procedures, loaded only when triggered | When the description matches |
| Local auto-memory (per machine, not versioned) | Pointers and per-machine facts only | Index every session |

### Why

- `CLAUDE.md` is loaded in full at every session: detail moved out of it into per-project files cut its
  size by about a third and keeps it far below the size limit.
- Everything lives in **one git repository**, so any workstation resumes the work with a single
  `git pull`. A memory that only lives locally does not achieve that.
- **First action of every session: `git pull` on that config repository** — it is the shared source of
  truth and a workstation can be behind.

### "Memory and stop" protocol

When the user says "save to memory" / "memory and stop", capitalize everything reusable from the
session, not only the project state:

1. Pull the config repository.
2. **Project state**: update the project file (decisions, pitfalls, progress, approved-but-unexecuted
   plans); for a split project, add the session to the **journal** then update the short file.
3. **Cross-project rules** go to `transverse-rules.md`.
4. **Reusable methods** (recipe, script, pitfall, measurement) go to the methods repository that fits,
   as tested code plus documentation (why, how to verify, rollback). Update, do not duplicate.
5. **Wiki**: the project wiki page(s), following the multi-platform process of section 7.
6. **Local memory**: add a pointer, fix or remove entries that became wrong.
7. Short imperative commit, `git pull --rebase`, push.
8. **Report**: a short list of what was capitalized and where — and what was *not* done, and why.
