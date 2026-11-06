---
name: omp-plugin-install-and-verify
description: "Install an OMP plugin from a GitHub marketplace and prove it actually loaded — covers the install commands, the verification ladder, and the false-negative traps (empty omp -p \"/cmd\" output, ask-the-model probes, stale in-session skill registry). Use when asked to install an OMP/OMP-compatible plugin or skill pack from a GitHub URL, or to verify such a plugin is working."
---

# OMP Plugin Install & Verify

Applies to Oh My Pi (`omp`) — not `pi`, not Claude Code. Check `which omp` first; `pi` may not exist.

## 0. Read the source repo's install instructions

Fetch `AGENTS.md` **and** `README.md` / `INSTALL.md` from the repo. `AGENTS.md` is the file map; `INSTALL.md`
carries the per-runtime command blocks. Find the **Oh My Pi (OMP)** section specifically — it differs from the
Pi, Claude Code, and Codex sections, and mixing them up is the most common failure.

## 1. Install (two commands, in order)

```bash
omp plugin marketplace add <owner>/<repo>
omp plugin install --scope user <plugin>@<marketplace>
```

The plugin id is `<plugin-name>@<marketplace-name>` — read both from the repo's marketplace manifest
(`marketplace.json`, `.claude-plugin/marketplace.json`, or `.agents/plugins/marketplace.json`).
They are often the same string (e.g. `i-have-adhd@i-have-adhd`), but read it rather than guessing.

`--scope user` installs to `~/.omp/`; omit it for the default. `--scope project` writes to `./.omp/`.

## 2. Verification ladder

Run these; each is cheap and each proves a different layer.

| # | Command / path | Proves |
|---|---|---|
| 1 | `omp plugin list` | registry entry exists, with version and scope |
| 2 | `omp plugin marketplace list` | marketplace source registered |
| 3 | `~/.omp/plugins/installed_plugins.json` | `installPath` present and real |
| 4 | `~/.omp/plugins/omp-plugins.lock.json` | `enabled: true` — gates `getEnabledPlugins()` |
| 5 | `ls -la ~/.omp/plugins/node_modules/` | symlink cache→node_modules (`#registerRuntimePlugin`) |
| 6 | `omp plugin doctor` | health check; expect `0 errors` |
| 7 | behavioral probe (§3) | the extension actually *executed* |

Steps 3–5 are the decisive ones. Steps 1 and 6 alone do **not** prove the extension loaded.

## 3. Behavioral probe — the only real proof

```bash
cd /tmp && omp -p "<prompt>" --no-session <plugin-flag>
```

Pick a signal the plugin emits and ask for it verbatim. For a plugin that injects a ruleset, ask it to quote the
first words of that ruleset; for one that adds a tool, ask it to call it.

**Control test**: an unknown flag hard-fails, so a *parsed* plugin flag is proof of autoload:

```bash
omp -p "ok" --definitely-not-a-flag --no-session   # → Error: unknown flag
```

That the plugin's own flag parses (instead of erroring) means the extension registered it at session start via
the ambient loader (`src/extensibility/extensions/loader.ts:686` → `getAllPluginExtensionPaths` →
`resolvePluginExtensionPaths`), with no explicit `-e` path. Autoload = proven.

Use `--no-session` so probes don't pollute session history.

## 4. Traps — all produce confident false negatives

1. **Empty `omp -p "/cmd"` output ≠ command not registered.** A registered slash command that takes no model
turn prints nothing. Never read silence as absence.
2. **Ask-the-model probes are unreliable for registration inventories.** A model asked "which commands do
extensions register" will report some and deny others that demonstrably exist. Ground truth is the extension
source (`pi.registerFlag` / `pi.registerCommand`) plus a behavioral differential.
3. **`skill://<name>` in the installing session fails even on a correct install.** The session's skill registry
loaded *before* the install. Verify in a **new** session. This is a load-order artifact, not a design limitation.
4. **`disable-model-invocation: true` does NOT block `skill://`.** The resolver reads `getActiveSkills()` with no
hide filter; `hide` only removes the skill from the model-facing prompt list. Don't cite it as the cause of a
`skill://` miss.
5. **This session can't use the plugin at all** if it started before the install. Say so; don't re-probe endlessly.
6. **`omp plugin describe` does not exist.** Valid actions: `install|uninstall|list|link|doctor|features|config|enable|disable|marketplace|discover|upgrade`.

## 5. Activation surfaces to report

Read the plugin's extension entry (`extensions/*.ts`) for what it actually registers, then report each:

- slash command(s) — `pi.registerCommand(...)`
- CLI flag(s) — `pi.registerFlag(...)`
- disable phrases — often handled in a `pi.on("input", ...)` hook (e.g. `stop <mode>`)
- always-on mechanism — commonly a flag file under `~/.omp/agent/` and/or a JSON config there

## 6. Stop conditions

Once §2 steps 1–7 pass, the install is done. Do not keep reading harness internals to "prove" autoloading
beyond the behavioral probe — that is already conclusive. Report install complete, state the activation
command, and tell the user to start a new session.
