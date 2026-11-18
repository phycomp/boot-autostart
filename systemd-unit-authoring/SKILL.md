---
name: systemd-unit-authoring
description: Use when authoring, reviewing, or debugging a systemd .service unit — choosing system vs user scope, Type=, Wants/After vs Requires, secrets in units, /run runtime dirs, one-shot prerequisites, or when a unit works by hand but fails after reboot or login. Triggers on service files, systemctl errors, and units that don't survive reboot.
---

# Systemd Unit Authoring

## Overview

A systemd unit is a **declarative contract**, not a shell script. Correctness comes from picking the right scope, Type, ordering edges, and minimal environment — then verifying the *installed* state. Get the scope wrong and nothing else matters.

Related: `setting-up-boot-autostart` covers the boot-registration decision and duplicate-instance guards. This skill covers the unit file itself.

## When to Use

- Writing or reviewing a `.service` unit.
- A unit works when run manually but fails after reboot or login.
- Debugging `systemctl` errors, wrong `Type=`, or ordering problems.

## Step 0 — Pick scope from an observable predicate

Do not guess. Ask/observe *when it should run*:

| Predicate | Scope | Unit location | WantedBy |
|---|---|---|---|
| "at boot, even if nobody logs in" | system | `/etc/systemd/system/` | `multi-user.target` |
| "when I log in, not at boot" | user | `~/.config/systemd/user/` | `default.target` |

A user unit cannot `sudo` (no TTY) and dies at last logout unless linger is enabled. `Before=`/`After=` only order units **within one manager** — a system unit cannot order before a user unit; use the cross-manager stamp handshake below. If the predicate is genuinely missing, state the assumption you're making and the one question you'd ask.

## Cross-manager dependency — the stamp handshake

When Step 0 puts the prerequisite in the system manager ("at boot") and the daemon in the user manager ("at login"), `Before=`/`After=` **cannot** connect them. A stamp file needs all three roles specified — saying only "use a stamp file" is not an answer:

- **Writer** (the system oneshot): remove any stale stamp at start (`rm -f /run/app-setup.done`), do the work, and create the stamp **only on success** at the very end (`install -m 0644 /dev/null /run/app-setup.done`). A non-zero exit then genuinely means "setup failed."
- **Location**: `/run/app-setup.done`. `/run` is tmpfs, so the stamp is wiped every boot — a stale success cannot survive into a new boot. Do not use a path you expect to persist.
- **Consumer** (the user daemon): choose **one** mechanism, never both in the same unit. Condition/Assert are evaluated **before** `ExecStartPre`, so if you put `AssertPathExists=` next to a wait loop, the Assert fails the job on a fast login and the wait loop never runs — turning a survivable race into a loud, non-retrying failure. For a boot-vs-login race use `ExecStartPre` below. For a stamp that should *already* exist (absence = real setup failure needing an operator), use `AssertPathExists=/run/app-setup.done` — preferred over `ConditionPathExists=`, which silently skips (`inactive`, `Result=success`). A false `Assert` fails the start job loudly with a journal message; **neither** moves the unit into the `failed` state.

Critical correction — **Condition/Assert do not interact with `Restart=`**. A skipped or assertion-failed unit never reaches `ExecStart`, so `Restart=on-failure` will **not** retry it. To ride out a boot-vs-login race, wait in `ExecStartPre` (an `ExecStartPre` failure *is* a real service failure and *does* feed `Restart=`):

```ini
[Service]
ExecStartPre=/bin/sh -c 'for i in 1 2 3 4 5 6 7 8 9 10; do [ -f /run/app-setup.done ] && exit 0; sleep 1; done; exit 1'
Restart=on-failure
RestartSec=2s
```

Choose deliberately: stamp absent because of a *race* → `ExecStartPre` retry; stamp absent because setup *genuinely failed* → loud `Assert` so an operator investigates.

## Output contract — your answer IS these parts, in order

Emit exactly these five parts. Do not dump every conceivable edge case; a bloated answer hides the decision.

1. **Scope decision** — the predicate, the choice, and any assumption/question.
2. **The unit file(s)** — complete text, absolute paths, correct specifiers.
3. **Install/enable/verify commands** — in order, including `daemon-reload`.
4. **Key-directive rationale** — 2–4 lines only: why `Type=`, why each `After`/`Wants`/`Requires`, how secrets and paths were handled.
5. **Cold-start caveat** — what you verified vs what only a real reboot/login proves.

## Key directives — compact reference

| Need | Directive | Note |
|---|---|---|
| Scope | (location + `WantedBy`) | see Step 0 |
| Main process | `Type=simple` | foreground process = MainPID; default |
| Daemonizes itself / writes pidfile | `Type=forking` + `PIDFile=` | only if it truly double-forks |
| Setup that must finish first | `Type=oneshot` + `RemainAfterExit=yes` | blocks transaction until exit |
| Home path | `%h` | **systemd does NOT expand `~` or `$HOME` in `ExecStart`** |
| Runtime dir (tmpfs `/run`) | `RuntimeDirectory=<name>` + `RuntimeDirectoryMode=` | recreated each start, removed on stop |
| Secret / env with special chars | `EnvironmentFile=` mode `0600` | see below — never inline in the unit |
| Ordering without hard failure | `Wants=` + `After=` | canonical for `network-online.target` |
| Hard prerequisite + ordering | `Requires=` **and** `After=` | `Requires` does not order; `After` does not require |
| Restart on crash | `Restart=on-failure` + `RestartSec=` | add `StartLimitBurst`/`StartLimitIntervalSec` to stop hot loops |
| Override a vendor unit | `foo.service.d/override.conf` | clear list with empty `ExecStart=` before re-adding |
| Logging | `StandardOutput=journal` + `SyslogIdentifier=` | filter with `journalctl -u` |

## Secrets — the one place agents rationalize

Put secrets in `EnvironmentFile=/etc/<app>/<app>.env` (mode `0600`, root-owned), **never** in the unit file (typically `0644`, world-readable).

The rationalization to reject: *"I'll inline `Environment="DB_URL=...password..."` but mention EnvironmentFile is safer for special chars"* — the unit is still world-readable; noting the safer option does not make inlining safe. If a value is a secret, the file mode is the deciding factor, not whether the characters are easy to escape. When in doubt, use `EnvironmentFile=`.

`%` is a specifier character — double it (`%%`) in a unit. Backslashes/quotes need escaping. An EnvironmentFile avoids the unit-escaping layer entirely.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Chose system vs user by guesswork | Use the login-vs-boot predicate (Step 0); state assumptions. |
| Used `~` or `$HOME` in the unit | Use `%h` (`%t`, `%S` for runtime/state). Verify with `systemctl show -p ExecStart`. |
| Inlined a secret in a `0644` unit | `EnvironmentFile=` mode `0600`; mentioning the alternative is not a fix. |
| `Requires=` alone, expecting ordering | Add `After=` — they are independent. |
| `After=` on `network-online.target` with no wait service | Enable the distro's wait-online service, or don't rely on it. |
| `ExecStart=` override without clearing inherited list | Add empty `ExecStart=` first, then the new one. |
| Forgot `daemon-reload` after editing | Reload, then `restart` — reload does not restart a running unit. |
| `systemctl start` assumed to mean boot-persistent | `start` ≠ `enable`; verify `is-enabled`. |
| oneshot re-running on every restart | Add `RemainAfterExit=yes`. |
| Condition/Assert used to "wait" for a stamp/file | They don't retry — failed check never reaches `ExecStart`; wait in `ExecStartPre` + `Restart=on-failure`. |
| Stamp "handshake" stated without mechanism | Specify writer (oneshot creates stamp on success), location (`/run`, tmpfs), and one consumer: race → `ExecStartPre` loop; genuine setup failure → `AssertPathExists`. Never both (Assert is checked first and blocks the loop). |
| Answer dumps 15 edge cases with no decision | Follow the 5-part output contract. |

## Verification (evidence, not assertions)

```sh
systemd-analyze verify <unit>                 # or --user verify for user units
systemctl daemon-reload                        # --user for user units
systemctl enable <unit>                        # boot/login persistence
systemctl start <unit>
systemctl is-enabled <unit>; systemctl is-active <unit>
systemctl show <unit> -p MainPID -p Result -p ExecMainStatus -p NRestarts
systemctl show -p ExecStart <unit>            # confirm %h expanded, no literal ~
journalctl -u <unit> -n 100 --no-pager
# env actually delivered (never store the secret in the unit just to check):
sudo cat /proc/$(systemctl show -p MainPID --value <unit>)/environ | tr '\0' '\n' | grep '^APP_'
```

Cold boot / fresh login and crash-after-restart behavior cannot be proven without a real reboot/login or a kill test — say what was and wasn't verified.
