---
name: omp-managed-skill-authoring-and-triggering
description: "Author, verify, and debug-trigger an OMP managed skill — including how automatic triggering actually works (description-only semantic matching, roster built at session start), why nested omp -p probes mislead, and the rule that a description must not poach another skill's vocabulary. Use when asked to create a new skill, why a skill does not trigger, or how to word a skill description."
---

# Authoring an OMP managed skill (and making it trigger)

Managed skills live at `~/.omp/agent/managed-skills/<name>/SKILL.md`. They are writable by `manage_skill` (`create`/`update`/`delete`); user-authored skills under `~/.omp/agent/skills` are never touched.

## 1. Scope the topic before writing

"Create a new skill" is under-specified — the topic is a real fork. Do not guess a topic and write a large artifact on it.

- If the user gave a topic, go straight to grounding.
- If not, ground candidate topics from the freshest in-flight work in the workspace (recent `git status`/`git log` across sibling repos, newest `docs/**/*.md`, in-progress branches), then ask ONE choice question with 3–4 concrete, evidence-backed candidates plus a "something else" escape. Report the evidence for each candidate in the option description.
- User-supplied answers may arrive as custom input rather than a picked option — treat that as the topic.

## 2. Ground every claim before writing, then re-verify the spans

A skill's currency is its facts. Writing it from memory produces a plausible, wrong artifact.

1. **Fan out read-only scouts** (cap 2 concurrent) over the repos the topic spans. Brief them with: exact target paths, the questions to answer, "report exact file:line spans and literal identifiers", "distinguish VERIFIED (read it in source, cite path:line) from INFERENCE", and "report the traps where the naive answer is wrong". No formatters, linters, or test suites.
2. **Read the nearest existing skills first** (`skill://<adjacent>`). Overlap is the main risk; the new skill must state its genuine delta, not restate a sibling.
3. **Reproduce the load-bearing numbers yourself** — do not take a scout's count. Scouts report stale or mis-anchored greps; e.g. a scout's "127 capabilities" came from an unanchored grep while `len(list_capabilities())` returned 128.
4. **After writing, spot-check every cited span** with `sed -n "${n}p" "$file"`. Drifted line numbers are the commonest defect — fix them. Also verify every literal identifier exists (`grep`); delete or replace any that does not.

## 3. Structure

- Frontmatter `description` is the entire discovery surface — see §5.
- Then: the core distinction up front (a table beating prose), an **Authoritative spans** table, a numbered **Procedure** with a copy-pasteable command, a **Traps** table (symptom → real cause → file:line), **Verification gates**, and **Related skills**.
- Embed enumeration/analysis scripts **inline as heredocs**, not as `/tmp` paths — a skill citing `/tmp/foo.py` dies on the next session.
- Traps beat prose: juniors and agents read tables.

## 4. Verify the skill by running it

Run the skill's own procedure verbatim as the acceptance test and confirm it reproduces the documented numbers. Then write the file out of the code fence into a temp script and run that too, to prove the fence itself is executable.

**Set `PYTHONDONTWRITEBYTECODE=1`** when the target repo tracks `.pyc` (check `git status` for modified `__pycache__`). Prove it with a before/after count:

```bash
git status --porcelain -- <dirs> | grep -c pyc   # run before and after; count must not increase
```

## 5. The description — and how triggering actually works

**Mechanism (verified on omp v18.2.10):**

- Only `name` + `description` from the frontmatter are surfaced. `~/.omp/agent/config.yml` has no `skills:` key; there is no keyword router. The roster is injected into context and the model matches your request against the descriptions by judgment.
- **The roster is built at session start.** A skill created mid-session does not auto-trigger until a later session begins.
- `skill://<name>` read is the **only deterministic path**.
- `omp --skills='<glob>'` / `--no-skills` **filter** the roster (they exclude all other skills) — they do not add one. So `omp -p --skills='X'` proves only that X's description matches a phrasing *once X is in the roster*; it never proves X wins against the full roster.
- Automatic triggering is therefore **probabilistic**. Say so rather than claiming proof.

**Consequence: write the description for the phrasing the user actually types.** Include the literal register of the request ("go through / walk through / enumerate each …"), not a summary of the implementation.

```yaml
description: "<what it does, in one clause>. Use when asked to <verbatim user phrasings>; <second phrasing>; <third>."
```

**Never adopt another skill's vocabulary.** A description that echoes a sibling's symptom or domain terms makes the two compete for the same prompt. When reviewing a new description, grep the sibling skills' descriptions and remove overlapping cues. Worked example: an enumeration skill kept cues on enumeration/inventory wording only, and dropped both *"chart renders but insight says insufficient data / missing inputs"* (owned by the conformance skill) and *"before binding a capability into the criteria docs"* (owned by the doc-authoring skill).

## 6. Testing a trigger without wasting sessions

- A nested `omp -p --skills='X' "<the task>"` runs a **full agent session** (2–3 min) and can time out *because it started doing the task* — a false negative that looks like a failure.
- Cheaper and more decisive: make the probe refuse to act, so it returns in seconds.

  ```bash
  omp -p --no-session --skills='X' \
    "Do NOT perform the task. In at most 20 words: name any skill from your available skill list that you would load for '<user phrasing>'."
  ```

- Run at most one such probe. More samples do not establish "what phrasing triggers this" — the mechanism is judgment-based.
- Report the `--skills=` limitation explicitly whenever you cite a probe result.

## 7. Cross-check against advisors skeptically

Review notes arrive as advisories. Adjudicate each against source before acting — they contradict each other.

- Verify every factual claim (line numbers, counts, "X lists N items"). Real example: an advisory asserted a 4-capability insight gap that did not exist; measurement showed all 18 web-allowlisted caps were profiled. Another asserted `omp --help` lists no skills flags, while it lists two.
- Accept claims that match evidence you already hold — e.g. "add `PYTHONDONTWRITEBYTECODE=1`" was confirmed by a `git status` you ran earlier showing modified `__pycache__` files.
- Soften overstated novelty claims yourself: if an existing sibling already carries facts you re-derived, say plainly what your delta is instead of claiming "not a duplicate".
