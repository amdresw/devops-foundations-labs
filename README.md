# devops-foundations-labs

Hands-on labs documenting my path towards a Junior DevOps / Cloud Engineer role. Each lab is a small, reproducible experiment: I state a hypothesis, run commands in a real environment, diagnose what happens and write down the root cause and what I learned.

These are learning labs, not production code. The goal is to show how I reason about systems, not just which commands I know.

## Environment

- Ubuntu 26.04.1 running in VirtualBox
- Labs are executed in the VM and documented from Windows 11 with VS Code and Git

## Labs

### Linux

| Lab | Topic | Skills |
| --- | ----- | ------ |
| [Lab 01](linux/troubleshooting/lab-01-systemd-exec-failure.md) | Diagnosing a failing systemd service (`status=203/EXEC`) | `systemctl`, `journalctl`, unit files, `daemon-reload` |
| [Lab 02](linux/troubleshooting/lab-02-systemd-daemon-reload.md) | What systemd runs after a unit file is edited (disk vs loaded vs running) | `systemctl show`, `NeedDaemonReload`, `ps`, `daemon-reload` |
| [Lab 03](linux/labs/lab-03-acl-directory-traversal.md) | Letting one group traverse a directory with an ACL, and the ACL mask | `setfacl`, `getfacl`, permissions, `sudo -u` |

More labs will be added here as they are completed. Networking and Git labs will get their own sections when they exist.

## How each lab is structured

1. **Objective**
2. **Environment**
3. **Initial hypothesis**
4. **Procedure**
5. **Commands and their purpose**
6. **Results**
7. **Root cause**
8. **Verification**
9. **Lessons learned**

## Learning path

Linux, Networking, Git, AWS, Docker, Terraform, CI/CD, observability and basic security. The repository grows in that order, and only with labs that are actually done.

## Notes

- Command output is trimmed to the relevant lines and checked for secrets before being committed.
- Labs may include open questions when something is not fully explained yet. I prefer documenting that honestly over inventing an explanation.
