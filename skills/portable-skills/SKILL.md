---
name: portable-skills
description: >
  How to author/refactor a Claude Code skill (or any helper script) so it works on ANY machine/OS,
  not just where it was written. Capability detection over hardcoded OS commands, layered secret
  lookup, no absolute paths, graceful degradation, declared deps. Use when creating a new skill,
  reviewing one, or making an existing skill run on another machine (Mac ↔ Linux server, etc.).
---

# Writing skills that travel

A skill's `SKILL.md` (the *knowledge*) is already portable — it's just instructions. Portability is
an **implementation** concern in whatever scripts the skill calls. The core mindset:

> **A portable skill asks "what can THIS machine do?" — it never assumes what the machine IS.**

Hardcoding `afplay`, `say`, `security`, `pbcopy`, `/Users/<you>/...` makes a skill silently break the
moment it runs somewhere else (e.g. synced to a Linux server). These patterns prevent that.

## The 7 patterns

**1. Capability detection, not hardcoded commands.**
Don't call `afplay`; find whatever exists.
```bash
for p in afplay mpg123 ffplay paplay cvlc; do command -v "$p" >/dev/null && PLAYER="$p" && break; done
```
```python
import shutil; player = next((p for p in ("afplay","mpg123","ffplay") if shutil.which(p)), None)
```

**2. Layered, OS-aware secret lookup.** Try sources in order, each guarded by "does this exist here?":
`env var` → macOS Keychain (`security …`, only if `which security`) → Linux libsecret
(`secret-tool …`, only if present) → a `.env` (portable locations) → **clear error listing what was tried.**

**3. No absolute machine paths.** `/Users/<you>/...` breaks instantly elsewhere. Use `$HOME` /
`Path.home()`, `~/.config/<skill>/`, `$XDG_CONFIG_HOME`, an env override, or paths **relative to the
script** (`dirname "$0"` / `Path(__file__).parent`). Keep an old absolute path only as the LAST
fallback layer, never the only one.

**4. Graceful degradation, never a crash.** If the final action can't run (headless, no audio sink,
missing tool), do the next-best thing and *say so*: "Saved to `/tmp/x.mp3` — no player found; install
mpg123." Exit cleanly with an actionable message, not a stack trace.

**5. Portable runtime for the logic.** Python/Node run everywhere; bash-with-macOS-builtins doesn't.
Put ALL the OS branching inside one helper script behind clean functions (`get_key()`, `play()`), so
the `SKILL.md` steps read the same on every machine.

**6. Declare deps + self-check.** State requirements in `SKILL.md`, and have the script check them up
front with an actionable message (e.g. "install mpg123: `sudo apt install mpg123`"). The script's
*runtime* is a dependency too (e.g. a `uv`-shebang script needs `uv` present on each box).

**7. Know what's inherently local.** Some skills can't travel and shouldn't pretend to — they SHOULD
say so in their description. Examples in this collection: `tts`/`tts-eleven` (need audio + a player on
the machine running them), anything reading the **macOS Keychain**, the Windows-Terminal setup notes,
macOS security commands (`spctl`, `fdesetup`, `socketfilterfw`). Flag these as Mac-only so it's
obvious which travel and which don't.

## Quick portability review checklist
- [ ] Any hardcoded OS command? (`afplay`, `say`, `pbcopy`, `open`, `security`) → wrap in detection.
- [ ] Any absolute path with a username? → make it `$HOME`-relative / script-relative / env-overridable.
- [ ] Secrets: does the lookup have ≥2 layers and a clear error? Does it cover both Mac & Linux stores?
- [ ] What happens on a headless box / when a tool is missing? → degrade + report, don't crash.
- [ ] Is the runtime (uv/node/python + libs) guaranteed on the target, or documented as a prereq?
- [ ] Does the `description:` honestly say if the skill is OS-specific?

## Worked example: `tts-eleven`
`tts-eleven/play.py` is the reference. It went from Mac-only (`afplay` + macOS Keychain + an absolute
`/Users/...` path) to cross-platform by applying #1–#4:
- **Key:** `get_api_key()` tries env → `_from_macos_keychain()` (guarded by `which security`) →
  `_from_linux_secret_tool()` (`secret-tool`, GNOME keyring) → `_from_dotenv()` over portable
  candidates (`$ELEVENLABS_ENV`, `~/.config/tts-eleven/.env`, next-to-script, legacy mac path) →
  a SystemExit that lists every source it tried.
- **Playback:** `play_or_save()` picks the first of `afplay/mpg123/ffplay/cvlc/mplayer` that exists;
  if none, it **keeps the mp3 and prints the path** instead of dying.
- **Result:** identical on the Mac (still afplay + .env), and it now runs on a Linux desktop (with a
  player + `uv` + a key source), degrading to "saved to …" on a headless box.

## TL;DR
Write the script to **probe** the machine — players, secret stores, paths — choose what's available,
and fall back loudly. The knowledge lives in `SKILL.md`; the *adaptability* lives in the helper.

## CLI hygiene — stdout, stderr, exit codes

A helper script that only ever `print()`s is fine while you're the only caller. It stops
being fine the moment something else consumes it — a pipe, a CI step, another skill.
(Source: freeCodeCamp, "CLI Development with Dart", 2026-09-24 — Dart-specific in the
article, but these three rules are language-agnostic.)

**Two streams, two jobs:**

| Stream | Carries | Rule |
|---|---|---|
| **stdout** | the actual result — the data the caller wants | nothing else, ever |
| **stderr** | progress, warnings, diagnostics, errors | never the payload |

The reason is concrete: a user redirecting your output to a file
(`tool > out.json`) must get a clean file, with errors still visible in the terminal.
Mixing the two silently corrupts the artifact. `git`, `curl` and `ffmpeg` all get this right.

```python
import sys
print(json.dumps(result))                      # stdout — the payload
print(f"fetched {n} rows", file=sys.stderr)    # stderr — the commentary
```

**Exit codes are the machine-readable half of the answer:**

| Code | Means |
|---|---|
| `0` | success (the default if you never call exit) |
| `1` | general failure |
| `2` | incorrect usage — wrong/missing arguments |

A non-zero exit stops a shell pipeline or a CI job immediately — which is exactly what
you want from a validation or quality-gate step. A script that fails loudly on stderr
but still exits `0` will be treated as a success by everything downstream.

⚠️ **This is the general form of the MCP stdio footgun** in `mcp-server-build`: an MCP
server over stdio uses stdout for JSON-RPC, so a stray `print()` corrupts the protocol.
Same rule, higher stakes — there, stdout isn't just the payload, it *is* the transport.

**Also worth keeping portable:** read config from environment variables with an explicit
fallback and a clear error when missing (never a silent default); detect the platform
rather than assuming; and handle a `None`/EOF from interactive input instead of crashing.

