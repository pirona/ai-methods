# How I Pilot Claude Code — My Method

A practical account of how I get consistent, high-quality results from Claude Code
across infrastructure, development, and creative projects.

---

## 1. The Foundation: CLAUDE.md as a Persistent Contract

The single most impactful thing I did was writing a detailed `CLAUDE.md` file
stored at the project (or global) level. It contains:

- **My role and context** — sysadmin, Linux/K8s, electronic music artist
- **Full infrastructure topology** — hypervisor, cluster, NAS, networking, services
- **Hard constraints** — HAProxy/TLS rules that must never be violated, ZFS rules, etc.
- **Tech stack reference** — so Claude never suggests tools outside my ecosystem
- **Numbered rules** — explicit, non-negotiable behaviors (no wildcard certs, no TLS bypass, etc.)

This means I never re-explain my environment. Claude enters every session already knowing
the full picture. The CLAUDE.md is a living document — I update it whenever a new hard
constraint emerges from real experience.

---

## 2. Memory System for Cross-Session Continuity

Claude Code has a persistent file-based memory system (`~/.claude/projects/.../memory/`).
I use it to store:

- **User profile** — who I am, how I like to communicate
- **Project state** — current bugs, in-progress work, key decisions
- **Feedback** — corrections and confirmed approaches from past sessions
- **References** — where to find things in external systems

This means a new session picks up exactly where the last one left off, without me
having to re-explain context.

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
- "ZFS: always add disks in pairs"
- "Every new service = dedicated cert → HAProxy frontend → backend → test"

These are derived from real incidents or strong personal constraints. Claude treats
them as non-negotiable, which eliminates an entire class of back-and-forth corrections.

---

## 6. Asking Before Long Work

For anything non-trivial, I ask Claude to outline the approach first:

> "how would you approach this?"

Claude gives a 2-3 sentence recommendation with the main trade-off. I redirect if needed,
then confirm before implementation starts. This prevents wasted effort on the wrong path.

---

## 7. Licensing and Documentation Rules (Baked In)

I added global rules to CLAUDE.md for things that would otherwise require constant reminders:

- **Apps**: always published under GNU GPL v3 — LICENSE file + SPDX header in sources
- **Docs**: English for public-facing content, French for personal wiki
  — Claude asks if the destination isn't specified

These rules travel with every project automatically.

---

## 8. Infrastructure Knowledge as First-Class Input

I don't ask Claude to "figure out" my infrastructure from context clues.
Every service, IP range, constraint, and convention is explicitly documented in CLAUDE.md.

The result: Claude's suggestions are immediately actionable, not generic.
A proposed HAProxy config already follows my exact cert naming convention.
A new service already includes the full deployment checklist.

---

## Key Takeaways

| What I do | Why it works |
|-----------|-------------|
| Detailed CLAUDE.md | Eliminates repetition, encodes hard constraints |
| Persistent memory | Cross-session continuity without re-explaining |
| Short imperative commands | Fast, no ambiguity |
| Ask for options on uncertain choices | Avoids rework on the wrong approach |
| Validate each step before the next | Tight feedback loop, errors caught early |
| Explicit numbered rules | No negotiation on critical constraints |
| Ask before long implementations | Alignment before effort |

---

*This document reflects my personal workflow as of mid-2026.
The method evolves — constraints get added, rules get refined from real incidents.*
