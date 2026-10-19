# Appendix — Shared skills we built together

A **skill** is a small, reusable procedure Claude Code loads *only when it is relevant*. It keeps long,
repeated procedures out of the always-loaded `CLAUDE.md` while making sure they are followed to the letter.
This appendix describes the pattern and the five skills that came out of real sessions. Everything below is
generic: replace the `<placeholders>` with your own hosts, repositories and token files.

---

## 1. Anatomy of a skill

```
skills/
├── link.sh                   # symlinks every skill into ~/.claude/skills (see below)
└── <skill-name>/
    ├── SKILL.md              # frontmatter + the procedure
    └── <helper script>       # optional, lives next to SKILL.md
```

`SKILL.md` starts with a frontmatter block, then a **short** body:

```markdown
---
name: <skill-name>
description: <What it does, and WHEN to use it — in the words you actually say. This line is the trigger.>
---

# <Title>

<Numbered steps, the commands, and the pitfalls. No essay.>
```

What we converged on after a few rounds:

1. **The `description` is the trigger.** Write it as "do X — use when the user says Y". A vague
   description means the skill never fires; an over-broad one fires when it should not.
2. **One skill = one procedure you repeat.** If you explained the same sequence twice in a session, it
   is a skill candidate.
3. **Short body (about 20-25 lines):** steps, exact commands, and the pitfalls you already paid for.
   Rationale lives in the project docs, not here.
4. **Pitfalls go in as soon as you hit them.** A skill is a living file: each incident becomes one line
   (for example "two merges in a row can answer 405 — wait 5 s and retry").
5. **Never put secrets in a skill.** Point to the token *file* (`~/.tokens/<service>_pat`), never to its
   content; pass tokens through a header, never in a URL or argument.
6. **Hard rules are repeated, not paraphrased.** If a skill touches something destructive, restate the
   rule ("never delete without explicit approval") inside the skill — it is the text Claude is reading at
   that moment.

### Sharing across workstations

Keep the `skills/` folder in the same git repository as your `memory/` files (section 21 of
`CLAUDE.md`), and link it after each `git pull`:

```bash
#!/bin/bash
# Links the shared skills of this repository into ~/.claude/skills (run on each workstation after git pull).
set -e
D="$(cd "$(dirname "$0")" && pwd)"
mkdir -p ~/.claude/skills
for s in "$D"/*/; do n=$(basename "$s"); [ -f "$s/SKILL.md" ] || continue; ln -sfn "${s%/}" ~/.claude/skills/"$n"; echo "linked: $n"; done
```

Verify: `ls -l ~/.claude/skills` shows symlinks into the repository. Rollback: remove the symlinks
(`rm ~/.claude/skills/<name>` only removes the link, not the repository folder).

---

## 2. The five skills

### 2.1 `memory-and-stop` — full capitalization at the end of a session

- **Trigger:** the user says "save to memory" / "memory and stop".
- **What it does:** runs the 8-step protocol of `CLAUDE.md` section 21 — project state (journal first for a
  split project), cross-project rules, reusable methods into the right methods repository, wiki, local
  memory pointer, commit/push, and a short report that also says what was **not** done.
- **Why a skill:** the protocol is long and easy to shorten under time pressure; as a skill it is applied
  the same way every time, and it is not in the permanent context.

```markdown
---
name: memory-and-stop
description: Full capitalization when the user says "save to memory" or "memory and stop": project state, methods, wiki, local pointer.
---
1. `git pull` the shared config repository.
2. Project state: update `memory/NN-*.md` (split project: add the session to the journal, then the short file).
3. Cross-project rules → `memory/transverse-rules.md`.
4. Reusable methods → the matching methods repo (tested code + docs: why, how to verify, rollback). Update, don't duplicate.
5. Wiki page(s) of the project.
6. Local memory: add a pointer, fix or remove wrong entries.
7. Short imperative commit, `git pull --rebase`, push (SSH).
8. Report: what was capitalized and where — and what was NOT done, and why.
```

### 2.2 `pr-ship` — branch → pull request → merge → deploy

- **Trigger:** "ship it", "merge and deploy", any PR on a self-hosted Gitea repository.
- **What it does:** branch from an up-to-date `main`, check (`node --check`, YAML parse, the repo's
  self-check, a real capture for UI changes), open the PR by API, merge, pull locally, deploy with
  `git pull --ff-only` on the target host, then verify the deployed files.
- **Rules learned the hard way:**
  - **Who merges?** By default the user. The assistant merges only after "merge and deploy" (or an explicit
    blanket go-ahead for the session); the authorization is valid for the session only.
  - Merge is `POST /repos/<owner>/<repo>/pulls/<n>/merge` with `{"Do":"merge"}`. A `POST` on `/pulls/<n>` answers 405.
    Closing a PR is `PATCH /pulls/<n>` with `{"state":"closed"}`.
  - **Two merges in a row:** the second can answer 405 and `mergeable` can stay `false` for a few seconds
    with no real conflict (`git merge-tree` shows none). Wait 5 s, retry, then check `"merged": true`.
  - Never delete a branch without approval. Restart a service only if its application code changed
    (static files need no restart) — and a restart needs approval too.

```markdown
---
name: pr-ship
description: Ship a change through Gitea: branch, PR via API, merge, then deploy with git pull on the target host. Use for any PR on a self-hosted repo or when the user says "merge and deploy".
---
1. `git checkout -b <type>/<topic>` from an up-to-date `main`; small commits, one PR = one job.
2. Verify (syntax check, self-check, real capture for UI). 
3. Push over SSH, then create the PR (token from `~/.tokens/<service>_pat`, header only):
   `curl -s -X POST -H "Authorization: Bearer $(cat <token-file> | tr -d '\n')" -H "Content-Type: application/json" \
     -d '{"title":"…","head":"<branch>","base":"main","body":"…"}' https://<git-host>/api/v1/repos/<owner>/<repo>/pulls`
4. Merge only after "merge and deploy": `POST …/pulls/<n>/merge` `{"Do":"merge"}`; on 405 wait 5 s and retry.
5. Deploy: `git pull` locally, then `ssh <deploy-host> 'cd <path> && git pull --ff-only'`.
6. Verify: `curl -s -o /dev/null -w '%{http_code}' http://<deploy-host>:<port>/static/<file>` → 200.
7. Never delete a branch without approval.
```

### 2.3 `cutout-triple-check` — cutting out an illustration cleanly

- **Trigger:** any cut-out of a character (ink drawing, manga, comic) from a white background.
- **What it does:** three controls before saying "done".
  1. **Measure:** flood-fill the background from the four borders, remove specks, erode the light fringe;
     the share of opaque pixels on each of the four borders must be **0**.
  2. **Numbered visual review:** overlay every enclosed white pocket with a number, zoom on each concavity
     (between arm and torso, elbow, between the legs, under a cape). For each: **air** (remove) or
     **matter** (skin, cloth, hair — on a black-and-white drawing skin is white too). Ambiguous cases are
     **flagged to the user, never decided silently**.
  3. **In-situ render:** magenta, light and dark backgrounds side by side, then the real theme. Look for a
     white halo, fake signatures, floating bits of ground, stains.
- **Lessons:** a cut-out cannot lengthen a figure that was cropped at generation — regenerate with margin
  (see section 18 of `CLAUDE.md`). White fur or hair with no outline can leak to the background and turn
  transparent: refill the leak, limited to a box. Keep the original pixels under the alpha channel so every
  step is reversible, and keep the previous version for rollback.

### 2.4 `comfyui-headless-job` — driving an image server without choking the machine

- **Trigger:** any render driven by script against a local node-based image generator.
- **Rules (full list in `CLAUDE.md` section 18):** one image = one job, never more than one of ours in the
  queue, always the `extra_pnginfo.workflow` metadata, fp8 text encoder, free models after each job, poll with
  retry on timeout (the server answers slowly while swapping), never interrupt or delete a job.
- **Practical split:** explore with the small fast model (about 20 s), render the final with the large one
  (minutes). If a job exceeds about 10 minutes the machine is swapping: check free RAM and *propose* a
  restart — do not do it unasked.

### 2.5 `cms-draft` — pushing a post to a blog as a draft

- **Trigger:** the user validates a Markdown article to send to the blog.
- **What it does:** a small script converts the Markdown to HTML and creates (or updates) the post through
  the Admin API — **always as a draft** — with feature image and every meta / social field filled.
- **Rules:**
  - Never publish or delete without explicit approval.
  - Feature image + `meta_title`/`meta_description` + social titles and descriptions, never `null`,
    written from the real content (`CLAUDE.md` section 20).
  - The body does not contain the H1 (the title is a separate field); no internal anchors (the CMS strips them).
  - **No wide tables:** the CMS makes them scroll sideways even inside an HTML card. One `###` section per
    step, with a code block and a "watch points" line, reads far better.
  - Re-read the post through the API after creation (status, meta, tags, number of headings and code
    blocks), then give the editing link.
  - A public post is written in English with generic examples (`example.internal`, `PROXY_IP`) — nothing
    from the real infrastructure.
  - The admin key lives in a `0400` file, never printed, never passed as an argument.

---

## 3. How a skill is born

1. **Notice the repetition** — the same sequence explained twice in a session, or a procedure that went
   wrong because a step was skipped.
2. **Capture it at "memory and stop"** rather than in the middle of the work: the pitfalls you just hit are
   fresh and become the most valuable lines.
3. **Write the shortest `SKILL.md` that would have prevented the mistake**, test it by invoking it on the
   next occurrence, and trim what Claude did not need.
4. **Feed it back:** when a new incident teaches something, add one line to the skill the same day.
5. **Keep it out of the skill** what the repository already records (code structure, git history),
   anything that only matters for one conversation, and any secret.
