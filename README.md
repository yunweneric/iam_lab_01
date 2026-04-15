# AlmaLinux 9 — Sudo Labs

Hands-on exercises for **granular sudo policy** and **sudo timestamp (password cache) behavior** on AlmaLinux 9. This README is organized so you can skim **what each lab proves**, then follow the **commands and evidence** in order.

---

## Table of contents

- [What this repo covers](#what-this-repo-covers)
- [Prerequisites](#prerequisites)
- [Lab 1 — Limited sudo by user](#lab-1--limited-sudo-by-user)
  - [1.1 Intended outcomes (who can run what)](#11-intended-outcomes-who-can-run-what)
  - [1.2 Create accounts](#12-create-accounts)
  - [1.3 Configure sudoers](#13-configure-sudoers)
  - [1.4 Verify as each user](#14-verify-as-each-user)
- [Lab 2 — Sudo timer (timestamp)](#lab-2--sudo-timer-timestamp)
  - [2.1 Concepts](#21-concepts)
  - [2.2 Grant sudo via `wheel`](#22-grant-sudo-via-wheel)
  - [2.3 Reset the timer manually](#23-reset-the-timer-manually)
  - [2.4 Disable the timer globally](#24-disable-the-timer-globally)
  - [2.5 Override for one user (Lionel)](#25-override-for-one-user-lionel)
  - [2.6 Inspect effective sudo rules](#26-inspect-effective-sudo-rules)
- [Takeaways](#takeaways)
- [Technologies](#technologies)
- [Screenshot index](#screenshot-index)

---

## What this repo covers

| Topic | You will |
| --- | --- |
| **Lab 1** | Create three users and assign **different sudo rules** (full sudo, one command, command alias). |
| **Lab 2** | Use **`wheel`**, **`sudo -k`**, **`Defaults timestamp_timeout`**, and **per-user Defaults** to control how often sudo asks for a password. |

---

## Prerequisites

- AlmaLinux 9 VM (or equivalent RHEL-family system using `sudo` and `visudo`).
- Ability to run commands as root or via an account that can use `sudo` for initial setup.
- Comfort with the shell: `su`, `sudo`, `useradd`, `passwd`, `usermod`, `systemctl`, `fdisk`, `iptables`.

---

## Lab 1 — Limited sudo by user

**Goal:** Show that sudo can be **narrow** (one binary), **alias-based** (`STORAGE`), or **broad** (full sudo).

### 1.1 Intended outcomes (who can run what)

After configuration, expect roughly this behavior:

| User | Role in this lab | Expected pattern |
| --- | --- | --- |
| **lionel** | Full sudo | Can run arbitrary commands with `sudo`, including `sudo su`. |
| **katelyn** | Single command | May run **`systemctl status sshd`** only; other `sudo` attempts should fail. |
| **maggie** | Alias `STORAGE` | May run commands under the `STORAGE` alias (e.g. disk tools); not full shell as root. |

### 1.2 Create accounts

Create three login accounts and set passwords:

```bash
sudo useradd lionel
sudo passwd lionel

sudo useradd katelyn
sudo passwd katelyn

sudo useradd maggie
sudo passwd maggie
```

Log in or `su` to each account to confirm the accounts work.

| Evidence | File |
| --- | --- |
| Optional: default session / context | [`screenshots/lab1-00-default-account.png`](screenshots/lab1-00-default-account.png) |
| Users created / verified | [`screenshots/lab1-01-user-accounts.png`](screenshots/lab1-01-user-accounts.png) |

### 1.3 Configure sudoers

1. **Edit safely** with `visudo` (never edit `/etc/sudoers` with a raw editor):

   ```bash
   sudo visudo
   ```

   | Step | File |
   | --- | --- |
   | Open `visudo` | [`screenshots/lab1-02-visudo.png`](screenshots/lab1-02-visudo.png) |

2. **Enable the `STORAGE` `Cmnd_Alias`** by uncommenting its line (remove leading `#`).

   | Step | File |
   | --- | --- |
   | `STORAGE` alias | [`screenshots/lab1-03-storage-alias.png`](screenshots/lab1-03-storage-alias.png) |

3. **Append user rules** at the end of the file (exact lines from the lab):

   ```text
   lionel ALL=(ALL) ALL
   katelyn ALL=(ALL) /usr/bin/systemctl status sshd
   maggie ALL=(ALL) STORAGE
   ```

   | Step | File |
   | --- | --- |
   | Rules added | [`screenshots/lab1-04-privilege-rules.png`](screenshots/lab1-04-privilege-rules.png) |

Save and exit `visudo` so syntax is checked automatically.

### 1.4 Verify as each user

**Lionel** — broad sudo:

```bash
su - lionel
sudo systemctl status sshd
sudo fdisk -l
exit
```

| Check | File |
| --- | --- |
| `systemctl status sshd` | [`screenshots/lab1-05-lionel-sshd.png`](screenshots/lab1-05-lionel-sshd.png) |
| `fdisk -l` | [`screenshots/lab1-06-lionel-fdisk.png`](screenshots/lab1-06-lionel-fdisk.png) |

**Katelyn** — only the approved `systemctl` line:

```bash
su - katelyn
sudo su
sudo systemctl status sshd
sudo systemctl restart sshd
sudo fdisk -l
exit
```

| Check | File |
| --- | --- |
| `sudo su` denied | [`screenshots/lab1-07-katelyn-sudo-su.png`](screenshots/lab1-07-katelyn-sudo-su.png) |
| `systemctl status sshd` | [`screenshots/lab1-08-katelyn-sshd-status.png`](screenshots/lab1-08-katelyn-sshd-status.png) |
| `restart sshd` denied | [`screenshots/lab1-09-katelyn-restart-sshd.png`](screenshots/lab1-09-katelyn-restart-sshd.png) |
| `fdisk` denied | [`screenshots/lab1-10-katelyn-fdisk.png`](screenshots/lab1-10-katelyn-fdisk.png) |

**Maggie** — `STORAGE` alias:

```bash
su - maggie
sudo su
sudo systemctl status sshd
sudo systemctl restart sshd
sudo fdisk -l
exit
```

| Check | File |
| --- | --- |
| `sudo su` denied | [`screenshots/lab1-11-maggie-sudo-su.png`](screenshots/lab1-11-maggie-sudo-su.png) |
| `systemctl status` denied | [`screenshots/lab1-12-maggie-sshd-status.png`](screenshots/lab1-12-maggie-sshd-status.png) |
| `restart sshd` denied | [`screenshots/lab1-13-maggie-restart-sshd.png`](screenshots/lab1-13-maggie-restart-sshd.png) |
| `fdisk` allowed (via alias) | [`screenshots/lab1-14-maggie-fdisk.png`](screenshots/lab1-14-maggie-fdisk.png) |

---

## Lab 2 — Sudo timer (timestamp)

**Goal:** Control **how long** sudo remembers your password, from **global defaults** to a **per-user** line. The lab uses the sample username **`taptue`** for the primary account; substitute your own where needed.

### 2.1 Concepts

- **Timestamp / timer:** After a successful `sudo` password, sudo may **not** ask again until a timeout elapses.
- **`sudo -k`:** “Kill” the timestamp — next `sudo` should **require a password** again (unless configured otherwise).
- **`Defaults timestamp_timeout`:** `0` means **always prompt** (no cache). Non-zero values are minutes.

### 2.2 Grant sudo via `wheel`

```bash
sudo usermod -aG wheel taptue
```

Then test several privileged commands as `taptue`:

```bash
sudo fdisk -l
sudo systemctl status sshd
sudo iptables -L
```

| Command | File |
| --- | --- |
| `usermod` / wheel | [`screenshots/lab2-01-wheel-group.png`](screenshots/lab2-01-wheel-group.png) |
| `fdisk -l` | [`screenshots/lab2-02-fdisk.png`](screenshots/lab2-02-fdisk.png) |
| `systemctl status sshd` | [`screenshots/lab2-03-sshd-status.png`](screenshots/lab2-03-sshd-status.png) |
| `iptables -L` | [`screenshots/lab2-04-iptables-list.png`](screenshots/lab2-04-iptables-list.png) |

### 2.3 Reset the timer manually

```bash
sudo fdisk -l
sudo -k
sudo fdisk -l
```

| Step | File |
| --- | --- |
| Before / after `sudo -k` | [`screenshots/lab2-02-fdisk.png`](screenshots/lab2-02-fdisk.png) (first run), [`screenshots/lab2-05-fdisk-after-sudo-k.png`](screenshots/lab2-05-fdisk-after-sudo-k.png) |

### 2.4 Disable the timer globally

Edit with `visudo`, find the **Defaults** area (search with `/Defaults`), and set:

```text
Defaults timestamp_timeout = 0
```

Re-test; each `sudo` should ask for a password:

```bash
sudo fdisk -l
sudo systemctl status sshd
sudo iptables -L
```

| Step | File |
| --- | --- |
| Defaults section | [`screenshots/lab2-06-defaults-section.png`](screenshots/lab2-06-defaults-section.png) |
| `timestamp_timeout = 0` | [`screenshots/lab2-07-timestamp-timeout-zero.png`](screenshots/lab2-07-timestamp-timeout-zero.png) |
| Password each time: `fdisk` | [`screenshots/lab2-08-fdisk-password-prompt.png`](screenshots/lab2-08-fdisk-password-prompt.png) |
| Password each time: `sshd` | [`screenshots/lab2-09-sshd-password-prompt.png`](screenshots/lab2-09-sshd-password-prompt.png) |
| Password each time: `iptables` | [`screenshots/lab2-10-iptables-password-prompt.png`](screenshots/lab2-10-iptables-password-prompt.png) |

### 2.5 Override for one user (Lionel)

Add a **per-user** Defaults line for Lionel, for example:

```text
Defaults:lionel timestamp_timeout = 0
```

| Step | File |
| --- | --- |
| Lionel-specific line | [`screenshots/lab2-11-defaults-lionel-line.png`](screenshots/lab2-11-defaults-lionel-line.png) |

Compare **`taptue`** vs **`lionel`** running the same three commands:

**As `taptue`:**

```bash
sudo fdisk -l
sudo systemctl status sshd
sudo iptables -L
```

| Command | File |
| --- | --- |
| `fdisk` | [`screenshots/lab2-12-taptue-fdisk.png`](screenshots/lab2-12-taptue-fdisk.png) |
| `sshd` | [`screenshots/lab2-13-taptue-sshd.png`](screenshots/lab2-13-taptue-sshd.png) |
| `iptables` | [`screenshots/lab2-14-taptue-iptables.png`](screenshots/lab2-14-taptue-iptables.png) |

**As `lionel`:**

```bash
su - lionel
sudo fdisk -l
sudo systemctl status sshd
sudo iptables -L
exit
```

| Command | File |
| --- | --- |
| `fdisk` | [`screenshots/lab2-15-lionel-fdisk.png`](screenshots/lab2-15-lionel-fdisk.png) |
| `sshd` | [`screenshots/lab2-16-lionel-sshd.png`](screenshots/lab2-16-lionel-sshd.png) |
| `iptables` | [`screenshots/lab2-17-lionel-iptables.png`](screenshots/lab2-17-lionel-iptables.png) |

### 2.6 Inspect effective sudo rules

```bash
sudo -l
```

| Step | File |
| --- | --- |
| `sudo -l` output | [`screenshots/lab2-18-sudo-l.png`](screenshots/lab2-18-sudo-l.png) |

---

## Takeaways

- **Least privilege:** Match each account to the smallest workable sudo rule (single command, alias, or full admin).
- **Safe editing:** Always use **`visudo`** for `/etc/sudoers` to catch syntax errors before they lock you out.
- **Aliases:** `Cmnd_Alias` groups commands so policies stay readable and consistent.
- **Timer policy:** `timestamp_timeout` can be tuned **globally** and refined with **`Defaults:username`** when one account needs stricter re-authentication.

---

## Technologies

- AlmaLinux 9
- `sudo`, `visudo`, `/etc/sudoers`
- `systemctl`, `iptables`, `fdisk`
- User tooling: `useradd`, `passwd`, `usermod`, `su`

---

## Screenshot index

All files live under [`screenshots/`](screenshots/). Names follow **`labN-XX-short-title.png`**.

### Lab 1

| File | Description |
| --- | --- |
| `lab1-00-default-account.png` | Optional context / default session |
| `lab1-01-user-accounts.png` | User accounts created or verified |
| `lab1-02-visudo.png` | Editing sudoers with `visudo` |
| `lab1-03-storage-alias.png` | `STORAGE` command alias enabled |
| `lab1-04-privilege-rules.png` | User privilege lines appended |
| `lab1-05-lionel-sshd.png` | Lionel: `systemctl status sshd` |
| `lab1-06-lionel-fdisk.png` | Lionel: `fdisk -l` |
| `lab1-07-katelyn-sudo-su.png` | Katelyn: `sudo su` not allowed |
| `lab1-08-katelyn-sshd-status.png` | Katelyn: allowed `systemctl status sshd` |
| `lab1-09-katelyn-restart-sshd.png` | Katelyn: restart denied |
| `lab1-10-katelyn-fdisk.png` | Katelyn: `fdisk` denied |
| `lab1-11-maggie-sudo-su.png` | Maggie: `sudo su` not allowed |
| `lab1-12-maggie-sshd-status.png` | Maggie: `systemctl` denied |
| `lab1-13-maggie-restart-sshd.png` | Maggie: restart denied |
| `lab1-14-maggie-fdisk.png` | Maggie: `fdisk` via `STORAGE` |

### Lab 2

| File | Description |
| --- | --- |
| `lab2-01-wheel-group.png` | User added to `wheel` |
| `lab2-02-fdisk.png` | `sudo fdisk -l` (baseline) |
| `lab2-03-sshd-status.png` | `sudo systemctl status sshd` |
| `lab2-04-iptables-list.png` | `sudo iptables -L` |
| `lab2-05-fdisk-after-sudo-k.png` | `fdisk` after `sudo -k` |
| `lab2-06-defaults-section.png` | Defaults section in `visudo` |
| `lab2-07-timestamp-timeout-zero.png` | `timestamp_timeout = 0` |
| `lab2-08-fdisk-password-prompt.png` | Password prompt on `fdisk` |
| `lab2-09-sshd-password-prompt.png` | Password prompt on `sshd` |
| `lab2-10-iptables-password-prompt.png` | Password prompt on `iptables` |
| `lab2-11-defaults-lionel-line.png` | `Defaults:lionel` line |
| `lab2-12-taptue-fdisk.png` | User `taptue`: `fdisk` |
| `lab2-13-taptue-sshd.png` | User `taptue`: `sshd` |
| `lab2-14-taptue-iptables.png` | User `taptue`: `iptables` |
| `lab2-15-lionel-fdisk.png` | User `lionel`: `fdisk` |
| `lab2-16-lionel-sshd.png` | User `lionel`: `sshd` |
| `lab2-17-lionel-iptables.png` | User `lionel`: `iptables` |
| `lab2-18-sudo-l.png` | `sudo -l` |

---

**Documentation note:** Screenshot paths in the body and the [Screenshot index](#screenshot-index) match the files in [`screenshots/`](screenshots/) using the `lab1-XX-*.png` and `lab2-XX-*.png` naming scheme.
