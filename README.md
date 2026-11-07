# dsh-cookie-hygiene

A cross-host agent skill for **DeepSeek Harness Desktop**: diagnose, mitigate and verify the
`dsh-auth-*` browser-session cookie overflow that eventually breaks startup with **HTTP 431** /
`desktop welcome: Web authentication failed`.

Created with the official `npx skills init` layout. It is **docs-only**: the repository ships
`SKILL.md` plus display metadata; every diagnostic and fix is a command block inside `SKILL.md`
that the agent runs. There are no bundled scripts and no dependencies.

## 1. What problem it solves

**Symptom.** DSH Desktop stops starting. The window says "DeepSeek Harness is unavailable" or
"The application could not start or stopped unexpectedly", and `crash-*-host.log` contains
`desktop welcome: Web authentication failed` or a mention of HTTP 431.

**Root cause chain.**

1. The desktop host starts its local web app with `--port 0`, so **every launch gets a random port**.
2. The browser session cookie is named `dsh-auth-` + `sha256(host:port)`, so a new port means a
   **new cookie name** - the old one is never overwritten.
3. The default lifetime is **30 days** (`connection.cookieMaxAgeDays`) and nothing ever prunes them.
4. Cookies ignore ports: every request to `127.0.0.1` carries **all** historical cookies.
5. One cookie costs about **226 bytes** in the `Cookie` header. Around the **68th** cookie the
   header exceeds Node's default 16 KB `maxHeaderSize`, the startup handshake is rejected with
   **431**, and the app cannot authenticate.

**Not this problem if** the account is logged out, the network is down, a plugin is broken, or
`dsh web` is used (web mode binds a fixed port, so the name is stable and reused).

The skill therefore answers one question first - *is this the bug or not?* - and only then offers
the fix. The real fix belongs upstream (name the cookie by host, or prune on start); this skill is
the mitigation and the verification path.

## 2. Main features

| Step | Capability | Writes anything? |
| --- | --- | --- |
| 1 | Locate the Electron cookie DB on Windows/macOS/Linux | No |
| 2 | Diagnose: exact count, per-cookie lifetime, header size, 16 KB share, verdict - or a size-based estimate while DSH runs | No |
| 3 | Attribute the failure to this bug, or rule it out | No |
| 4 | Cap new cookies' lifetime via a profile patch (`connection.cookieMaxAgeDays`, default 7 days), idempotent, YAML-validated | Yes, backed up |
| 5 | Clean history: backup, delete `dsh-auth-*`, `VACUUM`, keep the newest 10 backups, `--dry-run` | Yes, backed up |
| 6 | Generate a "clean then launch" wrapper so every cold start resets the jar | Yes, new file |
| 7 | Verify after restart: lifetime 7 days = patch loaded; count 1 = cleanup active | No |
| 8 | Roll back any of the above | - |

Properties:

- **Read-only first.** Diagnosis never writes; the agent reports a verdict before touching anything.
- **Reversible.** Every write is preceded by a backup; the patch is idempotent and can be removed.
- **Refuses to fight the lock.** While DSH runs, Chromium holds the DB exclusively; the skill
  reports that and switches to a size estimate instead of risking a corrupt DB.
- **Knows when it no longer applies.** If cookie names change or the DB is empty while the app
  runs, it reports "fixed upstream / not applicable" instead of forcing a patch.

## 3. Installation

Requirements

- **Node.js >= 22.5** (`node:sqlite` is used by the diagnostic and cleanup blocks).
- Access to the DSH Electron userData directory, e.g.
  `%APPDATA%\@deepseek-ai\dsh-desktop\Network\Cookies` on Windows.
- No admin rights, no network calls, no third-party packages.
- DSH must be **fully closed** (window and tray icon) for exact diagnosis and cleanup.
- Applies to **DSH Desktop 0.2.1-rc.x** (rc.3 / rc.5 confirmed). `dsh web` is not affected.

Install with the skills CLI (this repository contains `skills/dsh-cookie-hygiene`):

```bash
npx skills add <owner>/<repo>
```

Or copy the folder into an agent skills directory:

```bash
cp -r skills/dsh-cookie-hygiene ~/.agents/skills/
```

Hosts that read `agents/openai.yaml` get the display metadata (name, short description, brand
color, default prompt).

## 4. Usage

Three ways to use it:

**A. Just describe the symptom** and let the agent pick the skill:

> "DeepSeek Harness won't start. The window says 'unavailable' and the crash log mentions HTTP 431."
>
> "Check whether DSH's dsh-auth cookies have piled up again."

**B. Invoke it explicitly**: "Use `dsh-cookie-hygiene` to diagnose the current DSH install."

**C. Run the commands yourself**: copy the relevant block from
[`skills/dsh-cookie-hygiene/SKILL.md`](skills/dsh-cookie-hygiene/SKILL.md) - Step 2 for
diagnosis, Step 4 for the lifetime patch, Step 5 for cleanup, Step 6 for the wrapper.

Typical flow and side effects:

1. read-only diagnosis -> verdict (`safe` / `watch` / `danger` / `OVERFLOW`);
2. if overflow is plausible, the agent asks before writing;
3. patch: appends a `connection` block to `~/.dsh/profiles/desktop/cordis.patch.yml`
   (backup written next to it);
4. cleanup: writes `Network/Cookies.bak-<timestamp>`, deletes `dsh-auth-*` rows, `VACUUM`s the DB;
5. wrapper: creates a launcher next to the skill/user's location (the template is in `SKILL.md`);
6. verify after the next launch.

Rollback: restore `cordis.patch.yml.bak-*` (or delete the appended block); copy a
`Cookies.bak-*` back only while DSH is closed; stop using the wrapper.

## 5. Example input / output

### Input

> DSH desktop won't start. The window says "DeepSeek Harness is unavailable" and the crash log
> mentions `desktop welcome: Web authentication failed`.

### 5.1 Diagnosis, DSH running (size estimate only)

```powershell
$db = Join-Path $env:APPDATA '@deepseek-ai\dsh-desktop\Network\Cookies'
$f = Get-Item $db; $pages = [math]::Round($f.Length / 4096); "size=$($f.Length)B pages=$pages"
```

```
size=20480B pages=5
```

Pages map to an approximate cookie count: `5 -> 0-10`, `7 -> 11-20`, `8 -> 21-35`, `9 -> 36-45`,
`10 -> 46-55`, `11 -> 56-62`, `12 -> 63-70`.

### 5.2 Diagnosis, DSH fully closed (real output)

Severe case (pre-fix machine, 31 accumulated cookies):

```
count=31  header~7068B / 16384B (43.1%)  verdict=watch
lifetime distribution: 7d x2, 30d x29
7d => lifetime patch active; 30d => patch missing or not loaded
```

Healthy case (after patch + cleanup, one launch):

```
2026-11-07T13:00:19  life=7.0d  value=173B  dsh-auth-xxxxx...

count=1  header~228B / 16384B (1.4%)  verdict=safe
lifetime distribution: 7d x1
7d => lifetime patch active; 30d => patch missing or not loaded
```

### 5.3 Exact mode while DSH runs (expected refusal)

```
cannot open cookie DB - DSH is probably still running (exclusive lock): unable to open database file
quit DSH completely (including the tray icon) and retry; while it runs only the size estimate works
exit code: 2
```

### 5.4 Lifetime patch (idempotent)

First run:

```
patched: cookieMaxAgeDays = 7 (takes effect on next launch)
```

Second run:

```
already patched: a connection entry exists - inspect it, do not append a duplicate
```

### 5.5 Cleanup

Dry run:

```
dsh-auth-* cookies: 4
dry-run: nothing changed
```

Real run:

```
dsh-auth-* cookies: 4
backup: ...\Network\Cookies.bak-2026-11-01T08-34-36Z
deleted 4; file is now 20480B
backups kept: 10
```

### 5.6 Wrapper log (generated from the SKILL.md Step 6 template)

Illustrative only - the wrapper is generated by the agent for the user's platform, and its
messages may be localized:

```
[2026-11-08 00:44:58] launch wrapper
[2026-11-08 00:44:58] before: 1 dsh-auth-* cookie (lifetime 7d, header 1.4%) -> safe
[2026-11-08 00:44:58] clean: deleted 1, remaining 0, file now 20480B
```

## 6. Verified environment

- Windows 11 (build 26200), DeepSeek Harness Desktop 0.2.1-rc.5, Node.js 24.21.0.
- Verified against both cookie-value storage styles Chromium uses: plaintext in `value`, and
  encrypted in `encrypted_value`.
- `quick_validate.py` (from the skill-creator toolchain) reports **Skill is valid!**
- macOS / Linux paths and launchers are designed but **untested**.

## 7. Limitations

- Does not fix DSH itself: no changes to `app.asar`; the real fix belongs upstream.
- Only touches `dsh-auth-*` rows for loopback hosts. Cookie values are Chromium-encrypted and are
  never read or printed; the platform-login partition is untouched.
- While DSH runs the cookie DB is exclusively locked: no exact reads, no cleanup.
- Windows 11 does not allow programmatic taskbar pinning, and `.cmd`/`.bat` cannot be pinned at
  all - pin the generated shortcut once by hand.
- Only launches through the wrapper are cleaned. Updater-triggered relaunches, Start-menu entries
  and direct `exe` double-clicks bypass it; those are covered only by the shortened cookie lifetime.
- Keeping the app open longer than `cookieMaxAgeDays` days makes API requests fail with 401 until
  restart - a known cost of the short lifetime, not a new bug.
- No tray/GUI monitor: commands, logs and the wrapper only.

## 8. Repository layout

```
.
├── README.md
└── skills/
    └── dsh-cookie-hygiene/
        ├── SKILL.md            # procedure, with every command block inline
        └── agents/
            └── openai.yaml      # UI metadata for hosts that read it
```

## 9. License

MIT
