# Lab 02: What systemd runs after editing a unit file (daemon-reload)

## Objective

Find out what systemd uses when a unit file is edited on disk: its in-memory copy or the file itself. Separate three things that are easy to confuse: the file on disk, what systemd has loaded, and the process that is actually running.

This lab follows an open question from [Lab 01](lab-01-systemd-exec-failure.md).

## Environment

- Ubuntu 26.04.1 on VirtualBox, user `andres`
- Test unit: `/etc/systemd/system/lab-reload.service` (new file, deleted at the end)

## Hypotheses

- **A:** systemd keeps using its in-memory copy of the unit until `daemon-reload`.
- **B:** systemd re-reads the file on every `restart`.

Predictions made before running the experiment:

| Question | Prediction | Outcome |
| -------- | ---------- | ------- |
| `NeedDaemonReload` right after editing the file | `yes` (based on Lab 01) | Correct |
| `restart` without reload | "`ExecStart` changes at restart" | Wrong. Restart alone did not pick up the change |
| `show` and `ps` both 300 means A, both 600 means B | Logical decision table | Applied correctly to the results |
| `daemon-reload` without restart | `show` changes, the running process does not | Correct, confirmed with `ps` |

## Procedure

1. Created the unit with `ExecStart=/usr/bin/sleep 300`, ran `daemon-reload` and started it.
2. Edited the file to `sleep 600` with `sed`, without reloading, and compared `show` with the file.
3. Restarted without reloading.
4. Ran `daemon-reload`.
5. Repeated the cycle in the opposite direction (600 to 300), this time comparing `show` with `ps` at every stage.
6. Deleted the unit.

`ps` was missing from my first attempt: I typed the literal text `<MainPID>`, which bash treats as an input redirection, so the command never ran. Using `$(systemctl show -p MainPID --value <unit>)` lets the shell substitute the PID.

## Commands and their purpose

| Command | Purpose |
| ------- | ------- |
| `systemctl show <unit> -p ExecStart` | What systemd has **loaded** (its in-memory definition) |
| `systemctl show <unit> -p NeedDaemonReload` | Whether the file on disk differs from what is loaded |
| `systemctl show <unit> -p MainPID --value` | Print only the PID of the main process |
| `ps -p <PID> -o pid,cmd` | The command line the process is **really running** |
| `$(command)` | Command substitution: the shell runs the inner command and inserts its output |

## Results

**Stage 1: baseline (file 600, loaded 600)**

```text
PID CMD
5006 /usr/bin/sleep 600
```

**Stage 2: file edited to 300, `restart` without `daemon-reload`**

```text
Warning: The unit file ... changed on disk. Run 'systemctl daemon-reload' to reload units.
NeedDaemonReload=yes
ExecStart={ path=/usr/bin/sleep ; argv[]=/usr/bin/sleep 600 ; ... pid=5058 ... }

PID CMD
5058 /usr/bin/sleep 600
```

The PID changed (5006 to 5058), so the service really restarted, but it ran the **old** definition.

**Stage 3: `daemon-reload` without restart**

```text
NeedDaemonReload=no
ExecStart={ path=/usr/bin/sleep ; argv[]=/usr/bin/sleep 300 ; ... pid=0 ... }

PID CMD
5058 /usr/bin/sleep 600
```

systemd now has `300` loaded, while process 5058 still runs `600`.

An earlier run (same direction, 300 edited to 600) gave the same pattern: after the edit and before reloading, `show` still said `300` and `NeedDaemonReload=yes`; after `restart` without reload it still said `300`.

## Root cause

systemd works from its in-memory copy of the unit. Editing the file only changes the disk. `systemctl restart` restarts the process using the **loaded** definition. Only `daemon-reload` refreshes that definition, and even then a running process keeps its old command line until the next restart.

So three layers can disagree:

| Layer | Updated by |
| ----- | ---------- |
| File on disk | Editing the file |
| systemd's loaded definition | `daemon-reload` |
| The running process | `restart` (after the reload) |

Correct order after editing a unit: edit, `daemon-reload`, `restart`, then verify with both `show` and `ps`.

## Verification

Hypothesis A is supported by the `restart` without reload (stage 2): same old arguments with a new PID. The stage 3 result shows that `show` (loaded) and `ps` (running) are different views and can disagree.

## Lessons learned

- `NeedDaemonReload=yes` is the reliable signal of a stale loaded unit. A `changed on disk` warning also appears in `status` and `show` output when the loaded unit is stale, so its absence is informative too (it was missing in Lab 01).
- `show -p ExecStart` tells you what systemd intends to run; `ps` tells you what is running. Verify with `ps` when it matters.
- A `restart` is not enough after editing a unit: the change is only applied after `daemon-reload` and then another restart.
- Placeholders such as `<PID>` are not valid shell syntax when typed literally: `<` and `>` are redirections.
- A test that changes a value (300 to 600) is cleaner than one that breaks the service, because a status snapshot taken a few milliseconds after a failed start can mislead (see Lab 01).

## Open questions

1. **Lab 01 contradiction (resolved with a caveat).** In Lab 01, a `restart` without `daemon-reload` already ran the new, broken path `/usr/bin/slep`. I repeated that exact sequence on a fresh unit: the service stayed `active (running)` with `sleep 300`, `NeedDaemonReload=yes` and the `changed on disk` warning appeared (status taken 3 seconds after the restart). So the Lab 01 observation did not reproduce. The most likely cause is an unrecorded `daemon-reload` in Lab 01, supported by the missing warning there, but I cannot prove it.
2. **`NeedDaemonReload=yes` after a reboot.** After a real reboot (`uptime -s` and `journalctl --list-boots` confirmed a new boot), systemd showed `ExecStart` with `600`, equal to the file, but still reported `NeedDaemonReload=yes` plus the "changed on disk" warning. A possible cause is the VM clock (file modification time versus load time), but I did not verify it with `stat` and `date`.
