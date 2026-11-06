---
name: pr-description-from-branch
description: "Generate a PR description from the current branch's changes against its base branch. Use when asked to create or draft a PR description."
---

# PR Description from Branch

Generate a structured PR description comparing the current branch against its base target.

## Steps

1. **Identify the branch and base:**
   - `git branch --show-current` — current branch name
   - `git branch -r | grep -iE 'dev|main'` — find base candidates
   - Use `origin/dev` first if it exists, else `origin/main`. If neither, check GitHub default branch or ask.

2. **Get the changes (three-dot diff):**
   - `git diff origin/<base>...HEAD --shortstat` — file/insert/delete counts
   - `git diff origin/<base>...HEAD --diff-filter=AM --name-only` — added + modified files
   - `git diff origin/<base>...HEAD --diff-filter=DR --name-status` — deleted/renamed files (must check)

3. **Get commits:**
   - `git log --oneline --no-merges origin/<base>..HEAD` — commit list for the section
   - For richer context: `git log --no-merges origin/<base>..HEAD --format="%h %s%n%b"`

4. **Group changes by feature area** by scanning file paths, commit messages, and commit bodies. Look for patterns:
   - Module/directory prefixes (`app/Modules/X/`, `resources/js/Pages/X/`)
   - Commit subject keywords (`feat(sales-booster)`, `feat(event-application)`)
   - New files that indicate whole features (controllers, models, Vue pages, migrations)

5. **Produce a structured PR description with:**
   - Brief summary line
   - File/insert/delete stats
   - Table of feature areas (Status, What changed)
   - Bulleted breakdown per area
   - Commit list
   - Testing notes
   - Any caveats (merge commits, config files, etc.)

6. **Write to `PR_DESCRIPTION.md`** for the user to paste or reference.

## Key rules

- ALWAYS use three-dot diff (`origin/base...HEAD`) — two-dot can include upstream changes if base moved forward.
- ALWAYS check for deleted/renamed files — they don't appear in `--diff-filter=AM`.
- Group by feature, not by file type or commit — the reader cares about capability, not directory structure.
- Include the stat line (`X files changed, +Y / -Z`) prominently.
