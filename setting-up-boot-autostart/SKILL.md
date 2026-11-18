---
name: setting-up-boot-autostart
description: Use when a self-hosted service was down after a reboot because nothing restarts it, or when asked to make an app or its database auto-start on boot with systemd or cron. Covers reboot-fragile DB prerequisites (/run tmpfs, stale postmaster.pid) and avoiding a duplicate instance when a manual process is already live.
---

# Setting Up Boot Autostart

## Overview

Autostart is a **supervisor registration plus reboot-safe prerequisites**, not just a command. A foreground `app run` inside a shell dies at reboot and nothing brings it back. Register one boot hook, make the DB prerequisites idempotent, and guard against launching a second instance. Every safety claim must be **fail-closed**: if the guard cannot determine state, it must skip, never proceed.

## When to Use

- Service was healthy, then a reboot happened, and it never came back.
- Asked to "auto-start X on boot" for a self-hosted app that has a DB dependency.
- Config says one DB mode but the live process uses another (e.g. embedded vs external).

Not for: container orchestration (use the platform's restart policy), or a service already supervised by systemd/cron.

## Decide from an observable predicate, then commit

Check first: `ps -p 1 -o comm=`

| PID 1 | Mechanism | Registration |
|---|---|---|
| `systemd` | one systemd unit (+ optional tmpfiles drop-in) | `systemctl enable --now <unit>` — safe ONLY once the structural skip guard (below) is written; the guard makes `--now` harmless against a live instance |
| anything else | root `@reboot` cron + one idempotent boot script | `sudo crontab -e` |

**Commit to exactly one branch.** Do not present both designs as options or write files for both. State the observed PID-1 value and why the chosen branch follows.

## Required deliverable shape

Your output **is** these parts, in order. Each is a required slot, not a suggestion:

1. **Evidence line** — PID-1 value, whether sudo is non-interactive (`sudo -n true`), root cause line.
2. **One registration file** (+ at most one `tmpfiles.d` drop-in for a `/run` dir). Name it for the **service**, never for a tenant/company: if the user says "auto-start MyCompany", the file is `<app>.service`, not `mycompany.service`. If the user gives a literal unit name, use it exactly.
3. **A structural duplicate-instance guard** — see below. A prose note "don't start it now" is NOT this slot.
4. **DB prerequisites derived from the live system** — see below.
5. **Explicit minimal environment** — `WorkingDirectory=<app dir>`, `HOME`, full `PATH`, the DB URL. `npx --no-install` resolves the local package **relative to `WorkingDirectory`**; omitting it passes a checklist mention but fails to launch. Use `npx`, never `pnpm` (pnpm appends args to a shell string → injection risk).
6. **Restart policy + logging** — `Restart=on-failure`, backoff, log destination. `ExecCondition` is re-evaluated on every restart attempt; `on-failure` does not cover a hung-but-healthy process (add a watchdog only if that risk is real).
7. **Enable command + verification commands + an honest "cannot verify cold boot" statement.**

## Core patterns

Structural skip guard (makes a stray `start` harmless when a manual instance is live). Write this BEFORE running `enable --now`. It **fails closed**: if `curl` cannot be resolved, exit 1 makes systemd skip the unit — never treat "cannot check" as "unhealthy, go ahead and start".

```ini
# systemd: skip cleanly if :3100 is already healthy; proceed on boot.
# Missing curl => exit 1 => systemd SKIPS (safe side), never proceeds blindly.
ExecCondition=/usr/bin/bash -c 'CURL=$(command -v curl); [ -x "$CURL" ] || exit 1; "$CURL" -fsS --max-time 3 http://127.0.0.1:3100/api/health >/dev/null 2>&1 && exit 1 || exit 0'
```

For cron (only when PID 1 ≠ systemd), the equivalent is a `pgrep -f '<app> run'` / health-check guard around the launch.

DB prerequisites — derive, do not assume:

```sh
stat -c '%U:%G %a' <data_dir>              # who owns the data dir
ps -o user= -C postgres | sort -u         # which OS user actually runs PG
command -v pg_ctl; command -v npx; command -v curl   # real paths
cat <data_dir>/postmaster.opts            # the DB mode actually in use
```

Match **both** the PG runtime user **and** the app unit's `User=`/`Group=` to the data-dir owner — a `700` owner-owned dir and `npx --no-install` local resolution both break under `User=root`. `/run` is tmpfs: recreate its socket dir at boot via `tmpfiles.d` (`d /run/postgresql 0755 <user> <group> -`) **or** `RuntimeDirectory=` (pick one, not both). Clear a stale `postmaster.pid` **only if it is genuinely stale** — check liveness AND identity (a recycled PID fails the identity test):

```sh
p=$(head -n1 <data_dir>/postmaster.pid)
if kill -0 "$p" 2>/dev/null && readlink -f "/proc/$p/exe" 2>/dev/null | grep -q '/postgres'; then
  :   # genuinely alive postgres: keep the pidfile
else
  rm -f <data_dir>/postmaster.pid   # dead PID, or PID recycled by another process
fi
```

Never blanket `rm -f` — that can delete a live PG's pidfile, and `kill -0` alone is not enough. `pg_ctl start` often clears a dead pidfile itself; the guard is a fallback. Start PG idempotently behind `pg_isready`, then launch the app. Prefer **one unit** with `ExecStartPre` for the DB guard; split PG into its own unit only when it needs an independent lifecycle.

## Quick reference

**Use ONLY the column matching the branch you committed to via the PID-1 check.** Do not write out or "cover" the other column — doing so is the hedging failure from the Common Mistakes table.

| Need | systemd column | cron column (only if PID 1 ≠ systemd) |
|---|---|---|
| Skip if already healthy | `ExecCondition` health check | health/`pgrep` guard in script |
| Recreate `/run` dir | `tmpfiles.d` or `RuntimeDirectory=` | `install -d` at script top |
| Start DB if down | `ExecStartPre` + `pg_isready` | same inside boot script |
| Restart on crash | `Restart=on-failure` + `RestartSec` | periodic watchdog cron line |
| Minimal env | `Environment=` per var | `export` in script |

## Common Mistakes

| Mistake | Fix |
|---|---|
| "I'll write both designs so the operator picks", or filling both Quick-Reference columns | PID-1 check already decided it. Write one column, one registration file. |
| Assuming PG runs as `postgres` | `stat` the data dir + `ps -o user= -C postgres`; match the owner. |
| `User=root` (or any user ≠ data-dir owner) in the app unit | Set `User=`/`Group=` to the `stat` owner; a `700` dir and `npx --no-install` both break otherwise. |
| Unconditional `rm -f postmaster.pid`, or `kill -0` alone | Require liveness **and** `/proc/$p/exe` postgres identity; a recycled PID otherwise keeps a stale pidfile and PG fails to start. |
| Bare `curl` in the guard with an incomplete `PATH` | Derive `command -v curl` and fail closed (exit 1 → skip) if absent; "cannot check" must never mean "proceed". |
| "Don't start now" as a note | Encode it as `ExecCondition`/guard; procedure gets forgotten, structure doesn't. |
| `enable --now` before/without the skip guard | Write `ExecCondition` first; `--now` is safe only because the guard skips a live healthy instance. |
| Naming the unit `mycompany.service` / `econova.service` | Name it for the service. Tenants are DB rows; "EcoNova" in logs is not a company — don't invent it. |
| "Safe dry-run" / treating "test boot branch" as stop-and-start | Verify guards in isolation (closed-port curl, `pg_isready`); never execute the start path against a live service. |
| Two units + `Requires=` cascade when one suffices | One unit + `ExecStartPre`. |
| Trusting `config.json` DB mode over the live `DATABASE_URL` | `postmaster.opts` and the working manual command are authoritative; align config later, not during the fix. |
| `pnpm paperclipai` | Use `npx --no-install paperclipai` — pnpm appends args to a shell string (injection risk). |
| Omitting `WorkingDirectory` / relying on inherited `PATH` | Set both explicitly; `npx --no-install` resolves relative to `WorkingDirectory`. |
| `is-enabled`/verify before `daemon-reload` | Run `systemctl daemon-reload` first or you verify a stale unit. |
| Assuming `/run` survives reboot | Recreate the socket dir via `tmpfiles.d` or `RuntimeDirectory=` — `/run` is tmpfs. |

## Verification (evidence, not assertions)

- `systemctl daemon-reload` first, then: `systemd-analyze verify <unit>` → exit 0, or `crontab -l` shows the entry; `systemctl is-enabled <unit>` → `enabled`.
- Live PIDs unchanged, `pg_isready` accepting, `/api/health` still 200 after enabling.
- **Expected status after `enable --now` with a live instance:** `systemctl status` shows `inactive (dead)` / `Result: exec-condition` / "condition failed". This is SUCCESS — the guard correctly skipped. Do NOT delete or weaken the guard; confirm `is-enabled` is `enabled` instead.
- Simulate the minimal env: `env -i HOME=... PATH=... <DB_URL> npx --no-install <app> --version`.
- Test the guard's **boot branch** in isolation — first pick a port with NO listener (`ss -ltn | grep ':3101 '` → empty), point the health curl at it, confirm exit `0` (proceed), and start nothing:
  `bash -c 'curl -fsS --max-time 3 http://127.0.0.1:3101/api/health >/dev/null 2>&1 && exit 1 || exit 0'; echo $?` → `0`. A port that actually returns 200 would wrongly report "skip" and invite the guard to be removed.
- **Cannot verify cold boot without rebooting or stopping the live service.** Say so plainly; confirm at next boot with `systemctl status` (should be `active`) and a health check.

Tenants/companies are DB rows, not processes — "auto-start MyCompany" means server + DB up. Don't invent a missing tenant because its name appears in logs.
