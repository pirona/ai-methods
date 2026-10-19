# How I Pilot Claude Code — My Method

If you like it and/or useit, please consider a coffee ^ : 
→ [Ko-fi](https://ko-fi.com/billisdead)

A practical account of how I get consistent, high-quality results from Claude Code
across infrastructure, development, and creative projects.

Companion files: [`CLAUDE.md`](CLAUDE.md) (the full operator template, section 21 explains the memory
architecture) and [`APPENDIX-skills.md`](APPENDIX-skills.md) (the shared skills we built together).

---

## 1. The Foundation: CLAUDE.md as a Persistent Contract

The single most impactful thing I did was writing a detailed `CLAUDE.md` file
stored at the project (or global) level. It contains:

- **My role and context** — sysadmin, Linux/K8s, electronic music artist
- **Full infrastructure topology** — hypervisor, cluster, NAS, networking, services
- **Hard constraints** — HAProxy/TLS rules that must never be violated, storage and safety rules, etc.
- **Tech stack reference** — so Claude never suggests tools outside my ecosystem
- **Numbered rules** — explicit, non-negotiable behaviors (no wildcard certs, no TLS bypass, etc.)
- **Security by design** — hardening is a design constraint from day 1, not a post-deployment pass
- **Documentation philosophy** — doc is written from day 1, pedagogical, lives in the repo

This means I never re-explain my environment. Claude enters every session already knowing
the full picture. The CLAUDE.md is a living document — I update it whenever a new hard
constraint emerges from real experience.

---

## 2. Memory System for Cross-Session Continuity

Claude Code has a persistent file-based memory system (`~/.claude/projects/.../memory/`). I use it
for per-machine pointers, but the memory that matters is **versioned in git**, in the same repository
as my global `CLAUDE.md`, so any workstation resumes the work with a single `git pull`:

- **A short global `CLAUDE.md`** — profile, hard rules, and one index line per project. It is loaded in
  full every session, so its size matters: moving detail out of it cut it by about a third.
- **One memory file per project** (`memory/NN-slug.md`) — state, decisions, pitfalls, procedures. Loaded
  only when the task touches that project. Large projects get a short file plus a chronological journal.
- **A cross-project rules file** — collaboration, safety and image-generation rules, re-read before
  visual work, PRs or any destructive action.
- **Shared skills** — repeated procedures loaded on demand (see section 14 and the appendix).
- **Local auto-memory** — profile, feedback and references, kept as pointers.

The first action of every session is a `git pull` on that repository, and "memory and stop" triggers a
full capitalization protocol (project state, methods, wiki, local pointer). Details: `CLAUDE.md`, section 21.

---

## 3. Communication Style: Short, Imperative, Direct

I give Claude short, precise commands:

> "commit & push everything"  
> "restart the build"  
> "fix the QR code after EAS build"

No preamble. No politeness filler. Claude doesn't need it and neither do I.

When I want options rather than a decision imposed on me, I ask:

> "what are our options here?"

Claude lists trade-offs; I pick. This avoids wasted work on the wrong approach.

---

## 4. Iterative Validation Loop

My cycle is tight:

```
Instruction → Claude executes → I observe the result → Correct or confirm → Next step
```

I don't batch up multiple unrelated tasks. Each step is validated before moving on.
When something is wrong, I say so immediately and directly — no softening.
Claude adjusts and we continue.

---

## 5. Explicit Global Rules (No Ambiguity)

In my CLAUDE.md I encode rules as numbered, imperative statements:

- "Never disable TLS verification, even in dev"
- "Never suggest wildcard certs — one cert per subdomain"
- "Never kill or delete anything without explicit approval"
- "Every new service = dedicated cert → HAProxy frontend → backend → test"

These are derived from real incidents or strong personal constraints. Claude treats
them as non-negotiable, which eliminates an entire class of back-and-forth corrections.

---

## 6. Asking Before Long Work

For anything non-trivial, I ask Claude to outline the approach first:

> "how would you approach this?"

Claude gives a 2-3 sentence recommendation with the main trade-off. I redirect if needed,
then confirm before implementation starts. This prevents wasted effort on the wrong path.

### LLM Council for architecture and strategic decisions

For project conception and high-stakes decisions, I use the
[LLM Council skill](https://github.com/aiwithremy/claude-skills-llm-council) with Claude Desktop.
It convenes multiple models simultaneously and synthesizes their perspectives into a single answer —
useful for decisions where a single model's blind spots could lead the design astray.

Setup note: the skill triggers on English prompts only by default. I configured it to respond
in French to match my working language.

---

## 7. Licensing and Documentation Rules (Baked In)

I added global rules to CLAUDE.md for things that would otherwise require constant reminders:

- **Apps**: always published under GNU GPL v3 — LICENSE file + SPDX header in sources
- **Docs**: English for public-facing content, French for personal wiki
  — Claude asks if the destination isn't specified

These rules travel with every project automatically.

---

## 8. Security by Design — Baked In from Day 1

Security hardening is not a phase that comes after deployment — it's a design constraint
encoded in CLAUDE.md so it applies automatically to every project:

- Least privilege for users, services, and network rules from the start
- No open ports or `0.0.0.0` binds without documented justification
- Secrets management defined before the first commit (Vault, sealed-secrets, GitHub Secrets)
- TLS everywhere feasible, even on internal services
- Audit logging planned in the architecture phase, not retrofitted
- Threat model identified before proposing any architecture

The result: Claude's first proposal is already hardened. No "we'll secure it later" suggestions.

---

## 9. Documentation as a First-Class Deliverable

Documentation is written from day 1 and evolves alongside the code until completion.
The philosophy is baked into CLAUDE.md so it applies to every document Claude produces:

- **Simple**: clear language, no jargon without definition
- **Exhaustive**: prerequisites, architecture decisions, operational procedures, known limitations
- **Pedagogical**: explains *what* we do AND *why* — not just a list of commands to copy-paste

Every command block gets an explanation of what it does and what to verify afterward.
Architecture decisions are documented at the time they're made, in the same commit as the code.

---

## 10. Infrastructure Knowledge as First-Class Input

I don't ask Claude to "figure out" my infrastructure from context clues.
Every service, IP range, constraint, and convention is explicitly documented in CLAUDE.md.

The result: Claude's suggestions are immediately actionable, not generic.
A proposed HAProxy config already follows my exact cert naming convention.
A new service already includes the full deployment checklist.

---

## 11. The Execution Methodology — Locked In

Every session follows a fixed protocol, encoded in CLAUDE.md so it applies automatically.

**For vague or strategic questions**: LLM Council first (see section 6).

**For precise orders**, always in this order:
1. Review existing state
2. State the objective
3. Propose a plan (wait for approval before acting)
4. Identify integration points with the existing system
5. Implement the simplest, most maintainable solution

**During execution**: strictly sequential, one step at a time. No parallel actions without explicit approval.

**After each deployment**: code review of all interactions with the existing system.

**When "ok" is given**: final code review + project memory update.

The discipline here is the point: Claude proposes, I validate, Claude executes one step, I observe, we continue. No surprises.

---

## 12. Real-Device Validation Loop (Mobile Projects)

For mobile projects, a fix isn't "done" because the code review looks right — it's done
after Claude has reproduced the exact bug on a physical device and watched it actually go
away.

```
Fix hypothesis → Reproduce on real device (adb) → Confirm server-side result if relevant → Commit/tag
```

I connect Claude to my phone over adb wireless debugging (pairing code once, then
`adb connect` — the port rotates each session) and have it drive the actual screens: tap
through the real UI, type real input, trigger the real gesture that caused the bug. If the
fix touches anything server-side, Claude checks the persisted result, not just what the
screen shows — a screen can look fine while the save silently failed.

This caught me out once: a first fix shipped and built successfully in CI, but hadn't
actually been reproduced on-device before tagging. The bug was still there. A second pass,
this time with a real adb repro before touching the release tag, found the actual root
cause in a few minutes — something no amount of re-reading the code would have surfaced.
Now it's a hard rule in CLAUDE.md: no new build tag without an on-device repro first.

---

## 13. Safety Rules Learned From Real Incidents

Three rules were added after something actually went wrong, and they are non-negotiable:

- **Never kill, never delete without explicit approval** — files, workflows, cluster resources, posts,
  branches, jobs, processes; including test objects Claude created itself. Claude flags and proposes,
  it never executes on its own. Helper processes stop by themselves (`timeout`), they are not killed.
- **Thinking aloud is not a request to implement.** When I describe an example mid-design, Claude
  restates its understanding and asks "do you want me to implement this?" first. A 150-line feature coded
  from a passing example once had to be reverted entirely.
- **Feature PRs get a code review before merge** (correctness, security, interactions with what exists),
  reported to me before the merge is proposed. A pure fix is exempt.

---

## 14. Shared Skills — Procedures Loaded On Demand

Procedures I repeat (end-of-session capitalization, PR → merge → deploy, cleanly cutting out an
illustration, driving an image server, pushing a blog draft) became **skills**: short files Claude loads
only when their description matches what I ask. They live in the same git repository as my memory and are
symlinked into `~/.claude/skills` on each workstation. Every incident adds one line to the relevant skill,
so they get sharper over time.

The pattern and the five skills we built together are in [`APPENDIX-skills.md`](APPENDIX-skills.md).

---

## Key Takeaways

| What I do | Why it works |
|-----------|-------------|
| Detailed CLAUDE.md | Eliminates repetition, encodes hard constraints |
| Persistent memory | Cross-session continuity without re-explaining |
| Short imperative commands | Fast, no ambiguity |
| Ask for options on uncertain choices | Avoids rework on the wrong approach |
| Fixed execution methodology | Proposal → validation → sequential execution → review |
| Validate each step before the next | Tight feedback loop, errors caught early |
| Explicit numbered rules | No negotiation on critical constraints |
| Ask before long implementations | Alignment before effort |
| Security by design in CLAUDE.md | First proposal is already hardened, no retrofitting |
| Documentation philosophy in CLAUDE.md | Every doc produced is pedagogical by default |
| Real-device repro before tagging a fix (mobile) | Catches root causes a code review alone misses |
| Memory versioned in git, split per project | Any workstation resumes with one `git pull`; `CLAUDE.md` stays small |
| "Never kill or delete without approval" | Destructive mistakes become impossible by default |
| Shared skills + "memory and stop" | Repeated procedures are followed to the letter, and every session leaves the method sharper |

---

*This document reflects my personal workflow as of October 2026.
The method evolves — constraints get added, rules get refined from real incidents.*
