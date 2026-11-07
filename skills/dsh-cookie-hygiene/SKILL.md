---
name: dsh-cookie-hygiene
description: Diagnose and mitigate DeepSeek Harness Desktop auth-cookie accumulation. The desktop host binds each launch to a random port, so every start mints a new dsh-auth-* cookie (default 30-day lifetime, never pruned); after roughly 68 cookies the Cookie header exceeds Node's 16 KB limit and the startup handshake is rejected with HTTP 431 ("Web authentication failed"). Use this skill to diagnose read-only, confirm attribution, cap new cookie lifetime via a profile patch, clean history with backup and VACUUM, install a clean-then-launch wrapper, and verify after restart.
license: MIT
compatibility: DeepSeek Harness Desktop 0.2.1-rc.x; Windows primary (PowerShell 5.1), macOS/Linux designed but untested; Node.js >= 22.5 for node:sqlite; needs read access to the Electron userData cookie DB.
---

# DSH cookie hygiene

A procedural skill. It needs no bundled scripts: every step is a command the agent runs.
All write operations are reversible (backups first) and support a dry-run where possible.

## The failure this fixes

| Fact | Value |
| --- | --- |
| Cookie name | `dsh-auth-` + base64url(sha256(authority)), authority = `host:port` for HTTP |
| Desktop host port | `--port 0`, so a **new port every launch** -> a new cookie name every launch |
| Default lifetime | `cookieMaxAgeDays: 30`, no cleanup path anywhere |
| Cookie scope | host-only, `Path=/`, HttpOnly, SameSite=Strict; cookies ignore ports, so every request to any `127.0.0.1` port carries them all |
| Per-cookie size | ~226 bytes (52-byte name + ~173-byte signed value + `=`), plus 2 bytes separator |
| Hard limit | Node default `maxHeaderSize` = 16384 bytes -> **~68 cookies** overflow |
| Startup symptom | Electron welcome exchange uses `net.fetch(..., { credentials: "include" })`, so a 16 KB+ Cookie header returns 431 and the app throws `desktop welcome: Web authentication failed` (UI: "DeepSeek Harness is unavailable") |

Not this problem if: the account is logged out, the network is down, a plugin is broken, or `dsh web` is used
(web mode binds a fixed port, so the name is stable and overwritten).

## When to use

- DSH Desktop will not start, or the window shows "unavailable" / "stopped unexpectedly".
- `crash-*-host.log` mentions `Web authentication failed` or `HTTP 431`.
- An agent or user suspects unbounded `dsh-auth-*` growth.

## Verified environment

Windows 11 (build 26200), DeepSeek Harness Desktop 0.2.1-rc.5, Node.js 24.21.0.
The diagnostics were checked against both storage styles Chromium uses for cookie values
(plaintext in `value`, and encrypted into `encrypted_value`). macOS/Linux are untested.

## Step 1 - locate the cookie DB (read-only)

| OS | Path |
| --- | --- |
| Windows | `%APPDATA%\@deepseek-ai\dsh-desktop\Network\Cookies` |
| macOS | `~/Library/Application Support/@deepseek-ai/dsh-desktop/Network/Cookies` |
| Linux | `~/.config/@deepseek-ai/dsh-desktop/Network/Cookies` |

```powershell
$db = Join-Path $env:APPDATA '@deepseek-ai\dsh-desktop\Network\Cookies'
Test-Path $db; (Get-Item $db).Length
```

If it is missing, DSH has never started on this machine: not applicable.

## Step 2 - diagnose

### While DSH is running: size estimate only

Chromium holds the SQLite file with an exclusive lock, so the file cannot be opened. Estimate from size:

```powershell
$f = Get-Item $db
$pages = [math]::Round($f.Length / 4096)
"size=$($f.Length)B pages=$pages"
```

Pages -> approximate cookie count (calibrated against the same Chromium schema):
`5 -> 0-10`, `7 -> 11-20`, `8 -> 21-35`, `9 -> 36-45`, `10 -> 46-55`, `11 -> 56-62`, `12 -> 63-70`.
Multiply by ~228 B for a rough header size. Say clearly that this is an estimate.

### While DSH is fully closed: exact numbers

Write this to `%TEMP%\dsh-cookie-doctor.mjs` and run `node %TEMP%\dsh-cookie-doctor.mjs "<dbPath>"`:

```js
import fs from 'node:fs'
import path from 'node:path'
import { DatabaseSync } from 'node:sqlite'

const dbPath = process.argv[2] ?? path.join(
  process.env.APPDATA ?? path.join(process.env.HOME ?? '', '.config'),
  '@deepseek-ai', 'dsh-desktop', 'Network', 'Cookies')
const LIMIT = 16384
const EPOCH = 11644473600000
const num = (v) => (typeof v === 'bigint' ? Number(v) : v)

if (!fs.existsSync(dbPath)) { console.error('cookie DB not found: ' + dbPath); process.exit(1) }
let db
try {
  db = new DatabaseSync(dbPath, { readOnly: true, readBigInts: true })
  db.prepare('select count(*) as c from cookies').get()
} catch (e) {
  console.error('cannot open cookie DB - DSH is probably still running (exclusive lock): ' + e.message)
  console.error('quit DSH completely (including the tray icon) and retry; while it runs only the size estimate works')
  process.exit(2)
}
const rows = db.prepare(`select name, length(value) as vlen, length(encrypted_value) as enc,
  creation_utc as created, expires_utc as expires
  from cookies where name like 'dsh-auth-%' order by creation_utc`).all()
let bytes = 0
for (const r of rows) {
  const created = new Date(Math.round(num(r.created) / 1000) - EPOCH).toISOString().slice(0, 19)
  // Chromium timestamps are microseconds since 1601, so one day is 86400000000 us
  const life = (num(r.expires) - num(r.created)) / 86400000000
  // newer Chromium keeps the plaintext in `value`; older builds encrypt it into `encrypted_value` (+31B envelope)
  const enc = num(r.enc)
  const valueLen = Math.max(num(r.vlen), enc > 31 ? enc - 31 : 0)
  bytes += r.name.length + 1 + valueLen + 2
  console.log(`${created}  life=${life.toFixed(1)}d  value=${valueLen}B  ${r.name.slice(0, 18)}...`)
}
const pct = (bytes / LIMIT) * 100
const verdict = pct >= 100 ? 'OVERFLOW (431 at startup)' : pct >= 80 ? 'danger' : pct >= 40 ? 'watch' : 'safe'
console.log(`\ncount=${rows.length}  header~${bytes}B / ${LIMIT}B (${pct.toFixed(1)}%)  verdict=${verdict}`)
if (rows.length) {
  const lives = new Map()
  for (const r of rows) {
    const d = Math.round((num(r.expires) - num(r.created)) / 86400000000)
    lives.set(d, (lives.get(d) ?? 0) + 1)
  }
  console.log('lifetime distribution: ' + [...lives.entries()].sort((a, b) => a[0] - b[0]).map(([d, n]) => `${d}d x${n}`).join(', '))
  console.log('7d => lifetime patch active; 30d => patch missing or not loaded')
}
db.close()
```

## Step 3 - attribute

1. `count` grows roughly with launches and is not shrinking: accumulation confirmed.
2. Header share near/over 100%: this is the 431 cause; otherwise look elsewhere.
3. Lifetimes are all ~30 days: the lifetime patch is absent; ~7 days: patch is active but history is old.
4. Zero `dsh-auth-*` rows while the app is running and failing: not this issue (upstream may have changed the scheme).

## Step 4 - cap new cookie lifetime (profile patch, reversible)

Target `~/.dsh/profiles/desktop/cordis.patch.yml`. Idempotent, backs up first:

```powershell
$patch = Join-Path $env:USERPROFILE '.dsh\profiles\desktop\cordis.patch.yml'
$text = [IO.File]::ReadAllText($patch)
if ($text -match '(?m)^- id:\s*connection\s*$') {
  'already patched: a connection entry exists - inspect it, do not append a duplicate'
} else {
  Copy-Item $patch "$patch.bak-$(Get-Date -Format yyyyMMdd-HHmmss)"
  $add = "`r`n- id: connection`r`n  config:`r`n    cookieMaxAgeDays: 7`r`n"
  [IO.File]::WriteAllText($patch, $text + $add, (New-Object Text.UTF8Encoding($false)))
  'patched: cookieMaxAgeDays = 7 (takes effect on next launch)'
}
```

Rules:
- 7 days is the safe default. Steady state = `7 x launches per day`, so even 10 launches/day stays around 70 cookies only at the edge.
- Lower it (e.g. 3) if the user restarts often; raise it (e.g. 14) only if the app is kept running for weeks.
- The patch only changes the lifetime of **new** cookies; history must be cleaned separately (Step 5).
- If `connection` is not the node id anymore (DSH upgrade renamed it), the patch silently does nothing: detect it by re-diagnosing (lifetimes still 30 days) and report that upstream changed instead of guessing another id.

## Step 5 - clean history (backup + VACUUM + retention)

Only when DSH is fully closed. Dry-run first, then run without `--dry-run`.

```js
// dsh-cookie-clean.mjs <dbPath> [keepBackups=10] [--dry-run]
import fs from 'node:fs'
import path from 'node:path'
import { DatabaseSync } from 'node:sqlite'

const dbPath = process.argv[2]
const keep = Number(process.argv[3] ?? 10)
const dry = process.argv.includes('--dry-run')
if (!dbPath || !fs.existsSync(dbPath)) { console.error('missing cookie DB path'); process.exit(1) }

const stamp = new Date().toISOString().replaceAll(':', '-').replace(/\.\d+Z$/, 'Z')
const backup = `${dbPath}.bak-${stamp}`
let db
try {
  db = new DatabaseSync(dbPath)
  db.prepare('select count(*) as c from cookies').get()
} catch (e) {
  console.error('cannot open cookie DB - DSH is probably still running (exclusive lock): ' + e.message)
  console.error('quit DSH completely (including the tray icon) and retry')
  process.exit(2)
}
const before = db.prepare("select count(*) as c from cookies where name like 'dsh-auth-%'").get().c
console.log(`dsh-auth-* cookies: ${before}`)
if (dry) { console.log('dry-run: nothing changed'); db.close(); process.exit(0) }

if (!before) { console.log('nothing to clean'); db.close(); process.exit(0) }
fs.copyFileSync(dbPath, backup)
console.log('backup: ' + backup)
db.prepare("delete from cookies where name like 'dsh-auth-%'").run()
try { db.exec('pragma wal_checkpoint(truncate)') } catch {}
db.exec('vacuum') // deleting rows only frees pages; VACUUM shrinks the file
db.close()
console.log(`deleted ${before}; file is now ${fs.statSync(dbPath).size}B`)

const dir = path.dirname(dbPath)
const prefix = `${path.basename(dbPath)}.bak-`
const backups = fs.readdirSync(dir).filter((n) => n.startsWith(prefix)).sort().reverse()
for (const old of backups.slice(keep)) fs.unlinkSync(path.join(dir, old))
console.log(`backups kept: ${Math.min(backups.length, keep)}`)
```

Safety:
- Never run while DSH runs: opening the DB fails, and a forced write can corrupt it.
- Only rows whose name starts with `dsh-auth-` are touched; the platform-login partition is elsewhere and untouched.
- The cookie value is Chromium-encrypted; the skill never reads or prints it.

## Step 6 - prevent recurrence (clean-then-launch wrapper)

Generate a launcher that cleans before starting, so the jar is reset on every cold start:

```powershell
# launch-dsh.ps1 - if you add any non-ASCII text, save as UTF-8 **with BOM**
# (PowerShell 5.1 reads BOM-less files as GBK and mangles it)
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
$here = Split-Path -Parent $MyInvocation.MyCommand.Path
if (Get-Process -Name 'DeepSeek Harness' -ErrorAction SilentlyContinue) {
  # already running: launching again only focuses it; never touch the locked DB
  Start-Process 'D:\path\to\DeepSeek Harness.exe'; exit 0
}
& node (Join-Path $here 'dsh-cookie-clean.mjs') "<dbPath>" 10
Start-Process 'D:\path\to\DeepSeek Harness.exe'
```

- Call it from a `launch-dsh.cmd` (`powershell -NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File "%~dp0launch-dsh.ps1"`).
- Pin the shortcut manually: Windows 11 removed programmatic taskbar pinning, and `.cmd`/`.bat` cannot be pinned at all.
- Coverage: only launches through this entry are cleaned. Updater-triggered relaunches, Start-menu entries and direct double-clicks bypass it; the Step 4 lifetime cap bounds those.

## Step 7 - verify

1. Re-run the Step 2 doctor after the next launch. Expected: `count=1`, lifetime `7.0d`.
2. `count=1` + `7d` => both the wrapper and the patch work.
3. `count=1` + `30d` => cleanup works, patch did not load (re-check profile path / node id / DSH version).
4. `count` stays at 1 across several launches => every launch cleans (check the wrapper log).
5. Header share below a few percent => 431 cannot recur.

## Step 8 - rollback

- Patch: restore `cordis.patch.yml.bak-*`, or delete the appended `connection` block.
- Cleanup: copy a `Cookies.bak-*` back over the DB **only while DSH is closed** (normally unnecessary).
- Wrapper: stop using it and start the app directly; optionally delete the shortcut.

## Pitfalls (learned the hard way)

- The DB is **exclusively locked** while DSH runs: exact reads fail with "unable to open database file"; do not fight it.
- Deleting rows does not shrink the file; without `VACUUM` file-size estimates stay inflated.
- PowerShell 5.1 reads BOM-less UTF-8 scripts as GBK: a `.ps1` containing non-ASCII text must be written with a BOM.
- Capturing Node output from a `.ps1` that is launched by double-click needs UTF-8 console encoding, or the Chinese/UTF-8 text arrives as mojibake and string matching fails.
- `.cmd` / `.bat` cannot be pinned to the Windows 11 taskbar; point the shortcut at `powershell.exe` instead.
- Second launch while running only focuses the window (single-instance); it does not create a second cookie.
- Keeping the app open longer than `cookieMaxAgeDays` days makes API requests fail with 401 until restart: that is the cost of the short lifetime, not a new bug.
- Upstream may fix the root cause (cookie named by host, or pruning at start). If cookie names change or the DB stays empty while the app runs, report "not applicable / fixed upstream" instead of forcing a patch.
