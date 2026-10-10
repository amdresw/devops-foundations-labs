# Lab 01: Diagnosing a failing systemd service (status=203/EXEC)

## Objective

Create a small systemd service, break it in a controlled way, and diagnose the failure using only `systemctl status` and `journalctl`, without looking at the unit file. Then repair it, verify the fix and clean up.

## Environment

- Ubuntu 26.04.1 on VirtualBox, user `andres`
- Test unit: `/etc/systemd/system/lab-hello.service` (a new file; no existing service was touched)

## Initial hypotheses

Predictions made before running each step, and how they turned out:

| Step | Prediction | Outcome |
| ---- | ---------- | ------- |
| Start the healthy service | `Active: active` with the start time | Correct |
| Start the healthy service | Main PID "maybe 1", since systemd is the parent | Partly wrong. PID 1 is the *parent* (systemd). The Main PID is the process started by `ExecStart` (`sleep`, PID 4431) |
| After 300 s | The service "suspends" | Wrong. `sleep 300` simply exits with code 0 and the unit becomes `inactive (dead)`. Suspending (pausing) is a different thing |
| Broken service | `Active: inactive` | Wrong. It is `failed`. `inactive` means a normal stop, `failed` means something went wrong |
| Broken service | Bad path in `ExecStart` (`sleep` became `slep`) | Correct, and the log line proves it |
| Restart without `daemon-reload` | Service shows `active`, no warning at the top | The `active` result was misleading (see Results). The service did run the new path, which could not be reproduced in Lab 02, so this step most likely included a reload that I did not record |

## Procedure

1. Created `lab-hello.service` with `ExecStart=/usr/bin/sleep 300` and ran `systemctl daemon-reload`.
2. Started it and checked `status` (healthy baseline).
3. Broke it by changing `/usr/bin/sleep` to `/usr/bin/slep` with `sed`, then `daemon-reload` and `restart`.
4. Diagnosed with `systemctl status` and `journalctl -u lab-hello.service -n 20`.
5. Repaired the path, reloaded, restarted and verified.
6. Experiment: broke it again and restarted, intending to skip `daemon-reload`, then reloaded and restarted again. (See the open question: this step could not be reproduced.)
7. Restored the file, checked `NeedDaemonReload` before and after the reload, then deleted the unit.

## Commands and their purpose

| Command | Purpose |
| ------- | ------- |
| `sudo tee /etc/systemd/system/lab-hello.service` | Write the unit file with root privileges (`sudo echo > file` fails because the redirection runs in the unprivileged shell) |
| `systemctl daemon-reload` | Make systemd re-read unit files |
| `systemctl start` / `restart` / `stop <unit>` | Control the service |
| `systemctl status <unit>` | State, exit code, Main PID and last log lines |
| `journalctl -u <unit> -n 20` | Recent log history of the unit |
| `systemctl show <unit> -p NeedDaemonReload` | Is the unit file on disk different from what systemd has loaded? |
| `sed -i 's#old#new#' <file>` | Edit a file in place (used to introduce and revert the fault) |

## Results

**Healthy baseline**

```text
Active: active (running) since Fri 2026-10-09 10:35:20 -05
Main PID: 4431 (sleep)
CGroup: /system.slice/lab-hello.service
        └─4431 /usr/bin/sleep 300
```

**After the fault** (`/usr/bin/slep`)

```text
Active: failed (Result: exit-code) since Fri 2026-10-09 10:39:48 -05
Process: 4824 ExecStart=/usr/bin/slep 300 (code=exited, status=203/EXEC)

(slep)[4824]: lab-hello.service: Unable to locate executable '/usr/bin/slep': No such file or directory
(slep)[4824]: lab-hello.service: Failed at step EXEC spawning /usr/bin/slep: No such file or directory
systemd[1]: lab-hello.service: Main process exited, code=exited, status=203/EXEC
systemd[1]: lab-hello.service: Failed with result 'exit-code'.
```

**Restart (intended without `daemon-reload`)**: the first `status`, taken about 16 ms after the restart, showed `active (running)` with an empty CGroup and Main PID `(slep)`, while the log lines right below it already reported `status=203/EXEC`. After `daemon-reload` and another `restart`, the status showed `failed`.

**`NeedDaemonReload`**

```text
NeedDaemonReload=yes   (after editing the file, before reload)
NeedDaemonReload=no    (after daemon-reload)
```

**Cleanup**: after deleting the file and reloading, `status` showed `Loaded: not-found` but still `Active: failed`.

## Root cause

`ExecStart` pointed to a binary that does not exist (`/usr/bin/slep`). systemd forked the child process but it could not `exec` the program:

- Evidence: `Unable to locate executable '/usr/bin/slep': No such file or directory`
- Exit status `203/EXEC` is a systemd-specific code meaning "the binary could not be executed" (wrong path or missing `x` permission). It is different from a program that starts and then fails with its own exit code (for example `status=1/FAILURE`).
- The failing step is EXEC, after the fork. That is why a PID exists (4824) with the name in parentheses, `(slep)`: the process lived only for an instant.

## Verification

After restoring `/usr/bin/sleep 300` and reloading:

- `Active: active (running)`
- A new Main PID (7672) that shows `sleep`
- CGroup contains `/usr/bin/sleep 300`
- No error lines in the journal for the new start

The fault could also have been confirmed independently with `ls -l /usr/bin/slep /usr/bin/sleep` (the first should fail, the second should exist). I first tried `cd`, which is not a good check: `cd` only enters directories, so its failure alone does not prove the file is missing.

## Lessons learned

- `failed` and `inactive` are different states. Look for `failed` when troubleshooting.
- `203/EXEC` points at the path or the execute permission of the binary, not at what the program does.
- A single `status` taken milliseconds after a restart can show `active` for a service that has already failed. The log lines (or a second check) told the truth. Cross-check one view with another before concluding.
- The PID of the main process is not the PID of its parent. For a system service, the parent is systemd (PID 1).
- The `Loaded: ... static` line means the unit has no `[Install]` section, so it cannot be enabled at boot.
- `NeedDaemonReload` is a reliable way to know whether the unit on disk differs from the loaded one. A `changed on disk` warning also appears in `status` when the loaded unit is stale; its absence here is consistent with a reload having happened.
- Deleting a unit does not clear its failed state: the unit showed `Loaded: not-found` and `Active: failed`. `systemctl reset-failed <unit>` clears that state, but I did not run it in this lab.

## Follow-up and correction

In the "restart without `daemon-reload`" step, the service ran the new broken path. I could not reproduce that. In [Lab 02](lab-02-systemd-daemon-reload.md) I repeated the same sequence (edit to `slep`, `restart`, no reload) on a fresh unit: systemd kept running the old `sleep 300` and showed both `NeedDaemonReload=yes` and a `changed on disk` warning. Neither appeared in this lab.

The most likely explanation is that a `daemon-reload` was run in this lab without being recorded between the edit and the restart. That is an inference, not something I can prove from the output I kept. The conclusion to trust is the one from Lab 02: `restart` alone does not pick up an edited unit file.
