# Lab 03: Letting one group traverse a directory with an ACL

## Objective

Give the group `devops` access to a file inside a private directory, without letting it list that directory and without opening it to every user on the system. Understand how the ACL entries and the ACL mask interact with the traditional `ls -l` permissions.

## Environment

- Ubuntu 26.04.1 on VirtualBox
- Users: `andres` (owner), `juan` (member of `devops`), `nobody` (not a member)
- Group: `devops` (members `andres`, `juan`)
- Test tree: `/srv/lab-acl/` (new, deleted at the end). No existing directory was modified.

```text
/srv/lab-acl/outer/            andres:andres  drwx------   (private)
/srv/lab-acl/outer/inner/      andres:devops  drwxr-x---
/srv/lab-acl/outer/inner/file.txt  andres:devops  -rw-r-----
```

## Hypotheses

Predictions made before running each step:

| Question | Prediction | Outcome |
| -------- | ---------- | ------- |
| `juan` reads `file.txt` before the ACL | Fails: `juan` has no `x` on `outer` | Correct. Confirmed by removing the ACL entry and retesting (see Results) |
| Minimal `chmod` fix | "Give all permissions of the directory" | Too broad. The minimum is `x` only (`o+x`), which still has a side effect: it opens traversal to every user |
| ACL `getfacl` entries | `devops` and `andres`, doubting the owner entry | Right to doubt: `user::` is the owner, `group::` the owning group, `group:devops:` is the new entry |
| `ls -ld` after the ACL | Shows `x` for `devops` | The `x` shown is the **mask**, not the `devops` entry. The visible change is the `+` at the end |
| `juan` cat, `ls outer`, `ls inner` after the ACL | cat works, both `ls` fail on `outer` only | Correct for the three tests |
| Mask set to `---` | `devops` effective permission is `---` and `juan` cannot read | Correct |
| `nobody` reads the file | Denied | Correct |
| `andres` enters `outer` with mask `---` | Yes, owner entry is not limited by the mask | Correct. `ls /srv/lab-acl/outer` listed `inner` while the mask was `---` |

## Procedure

1. Checked the starting point (`id juan`, `getent group devops`, `getfacl --version`).
2. Built the tree above with `mkdir`, `chown`, `chgrp` and `chmod`.
3. Set an ACL on `outer` only: `setfacl -m g:devops:x /srv/lab-acl/outer`.
4. Tested as `juan` with `sudo -u juan`, which starts a fresh process with `juan`'s current groups.
5. Lowered the ACL mask to `---` to see its effect, then restored it to `--x`.
6. Ran a control test as `nobody`, a user outside the group.
7. Removed `/srv/lab-acl/`.

## Commands and their purpose

| Command | Purpose |
| ------- | ------- |
| `setfacl -m g:devops:x <dir>` | Modify the ACL: give group `devops` only the `x` permission |
| `getfacl <path>` | Show the full ACL, including the mask and effective permissions |
| `setfacl -m m::<perms> <dir>` | Set the ACL mask |
| `sudo -u <user> <command>` | Run one command as another user, to test their access |
| `ls -ld <dir>` | Show the directory itself; a trailing `+` means it has an ACL |

## Results

**ACL after `setfacl -m g:devops:x`**

```text
# file: srv/lab-acl/outer
# owner: andres
# group: andres
user::rwx
group::---
group:devops:--x
mask::--x
other::---
```

```text
drwx--x---+ 3 andres andres 4096 Oct 20 10:57 /srv/lab-acl/outer
```

**Tests as `juan` with the ACL in place**

```text
$ sudo -u juan cat /srv/lab-acl/outer/inner/file.txt
contenido de prueba
$ sudo -u juan ls /srv/lab-acl/outer
ls: cannot open directory '/srv/lab-acl/outer': Permission denied
$ sudo -u juan ls /srv/lab-acl/outer/inner
file.txt
```

**Mask set to `---`**

```text
group:devops:--x		#effective:---
mask::---
```

```text
$ sudo -u juan cat /srv/lab-acl/outer/inner/file.txt
cat: /srv/lab-acl/outer/inner/file.txt: Permission denied
```

**Mask restored to `--x`: `juan` versus `nobody`**

```text
$ sudo -u juan cat /srv/lab-acl/outer/inner/file.txt
contenido de prueba
$ sudo -u nobody cat /srv/lab-acl/outer/inner/file.txt
cat: /srv/lab-acl/outer/inner/file.txt: Permission denied
```

**Baseline without the `devops` entry** (removed with `setfacl -x g:devops`, then restored)

```text
$ sudo -u juan cat /srv/lab-acl/outer/inner/file.txt
cat: /srv/lab-acl/outer/inner/file.txt: Permission denied
$ sudo -u juan ls /srv/lab-acl/outer
ls: cannot open directory '/srv/lab-acl/outer': Permission denied
```

**Owner with the mask at `---`**

```text
$ ls /srv/lab-acl/outer
inner
```

## Root cause (why it behaves this way)

To reach a file, a user needs `x` on **every** directory in the path. `outer` was `drwx------`, so `juan` fell into the "others" class and had nothing there. The ACL entry `group:devops:--x` adds one exception for one group, without changing the owner or the owning group of `outer`.

Because the entry has only `x` and no `r`, `devops` can pass through `outer` to a path it already knows, but cannot list its contents. The permissions on `inner` and `file.txt` (group `devops`, `r-x` and `r--`) then decide what happens next.

The mask caps the permissions of all group entries and named entries, so an entry can look correct in `getfacl` and still be ineffective. `getfacl` tells this with `#effective:`.

## Verification

- `juan` (in `devops`) reads the file: works.
- `juan` lists `outer`: denied, as intended.
- `nobody` (not in `devops`) reads the file with the same ACL: denied. This shows the exception is limited to the group.
- With the mask at `---` `juan` is denied again even though the entry still says `--x`.

## Why not `chmod o+x`

Reasoned, not run in this lab: `chmod o+x outer` would also let `juan` through, but `nobody` too, and every other local user. The ACL keeps the exception limited to `devops`. Another option without ACL is `chgrp devops outer` plus `chmod 710`, but that replaces the owning group of `outer`, which is not appropriate for something like a home directory.

## Lessons learned

- A missing `x` on any directory of the path produces `Permission denied`, not "not found".
- `x` on a directory (traverse) and `r` (list) are independent.
- When a directory has an ACL, the group triplet in `ls -l` shows the **mask**, not the owning group's permissions. Use `getfacl`.
- The mask can silently disable named ACL entries. Look for `#effective:` when something that should work does not.
- Test with a user inside the group and a control user outside it. The control is what proves the permission is scoped.
- The mask limits group and named entries but not the owner (`user::`) or `other::`: the owner kept access with the mask at `---`.
- `setfacl -x g:devops <dir>` removes a single ACL entry without touching the rest, which makes it easy to compare before and after.
