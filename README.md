# RHEL 9 – Password & Account Security Policy Playbook

This Ansible playbook enforces password complexity, account lockout, password aging, and session security policies on **Red Hat Enterprise Linux 9** servers.

---

## 📋 Table of Contents

- [Target Hosts](#target-hosts)
- [Project Structure](#project-structure)
- [Policies Applied](#policies-applied)
  - [1. Password Complexity](#1-password-complexity)
  - [2. Account Lockout (Faillock)](#2-account-lockout-faillock)
  - [3. Password Aging](#3-password-aging)
  - [4. Password Length](#4-password-length)
  - [5. Password History](#5-password-history)
  - [6. PAM Faillock (system-auth)](#6-pam-faillock-system-auth)
  - [7. Session Logout (KillUserProcesses)](#7-session-logout-killuserprocesses)
  - [8. Login Timeout](#8-login-timeout)
- [Variables Reference](#variables-reference)
- [Usage](#usage)
- [Running Specific Sections](#running-specific-sections)
- [Notes](#notes)

---

## Target Hosts

The playbook applies to all hosts defined in the `inventory` file:

| Host          |
|---------------|
| 10.10.20.99   |
| 10.10.20.98   |
| 10.10.20.97   |
| 10.10.20.96   |

> A pre-task assertion verifies each target is running **RHEL 9** before making any changes.

---

## Project Structure

```
RHE9-PassPolicy/
├── README.md                                          # This file
├── inventory                                          # Target hosts
├── rhel9-password-policy.yml                          # Main playbook
└── roles/
    └── password_policy/
        ├── defaults/
        │   └── main.yml                               # Default variables
        ├── handlers/
        │   └── main.yml                               # Service restart handlers
        ├── tasks/
        │   └── main.yml                               # All policy enforcement tasks
        └── templates/
            ├── pwquality.conf.j2                      # Password complexity template
            └── faillock.conf.j2                       # Faillock configuration template
```

| Component | Purpose |
|-----------|---------|
| **Playbook** (`rhel9-password-policy.yml`) | Entry point — targets hosts and invokes the role |
| **Role** (`roles/password_policy/`) | Contains all tasks, variables, templates, and handlers |
| **Defaults** (`defaults/main.yml`) | All tunable variables with default values |
| **Tasks** (`tasks/main.yml`) | The 8 security policy enforcement tasks |
| **Handlers** (`handlers/main.yml`) | Restarts `systemd-logind` when config changes |
| **Templates** (`templates/`) | Jinja2 templates for `pwquality.conf` and `faillock.conf` |

---

## Policies Applied

### 1. Password Complexity

**File modified:** `/etc/security/pwquality.conf`

Enforces strong password requirements via the `pam_pwquality` module.

| Parameter     | Value | Description                                       |
|---------------|-------|---------------------------------------------------|
| `minlen`      | 12    | Minimum password length                           |
| `minclass`    | 4     | Minimum number of character classes required      |
| `dictcheck`   | 1     | Reject passwords found in dictionary              |
| `dcredit`     | -1    | Require at least 1 digit                          |
| `ucredit`     | -1    | Require at least 1 uppercase letter               |
| `lcredit`     | -1    | Require at least 1 lowercase letter               |
| `ocredit`     | -1    | Require at least 1 special character              |
| `maxrepeat`   | 1     | Maximum consecutive identical characters allowed  |

---

### 2. Account Lockout (Faillock)

**File modified:** `/etc/security/faillock.conf`

Configures automatic account lockout after repeated failed login attempts.

| Parameter        | Value | Description                                                   |
|------------------|-------|---------------------------------------------------------------|
| `deny`           | 6     | Lock account after 6 failed attempts                          |
| `fail_interval`  | 900   | Time window (seconds) in which failures are counted (15 min)  |
| `unlock_time`    | 300   | Auto-unlock after 300 seconds (5 min)                         |

---

### 3. Password Aging

**File modified:** `/etc/login.defs`

Controls how often users must change their passwords.

| Parameter       | Value | Description                                    |
|-----------------|-------|------------------------------------------------|
| `PASS_MIN_DAYS` | 1     | Minimum days before a password can be changed  |
| `PASS_MAX_DAYS` | 90    | Maximum days a password remains valid          |

---

### 4. Password Length

**File modified:** `/etc/login.defs`

Sets the minimum and maximum password length at the login level.

| Parameter      | Value | Description               |
|----------------|-------|---------------------------|
| `PASS_MIN_LEN` | 12    | Minimum password length   |
| `PASS_MAX_LEN` | 12    | Maximum password length   |

---

### 5. Password History

**File modified:** `/etc/pam.d/system-auth`

Prevents password reuse by appending `remember=4` to the `pam_unix.so` line.

| Parameter  | Value | Description                                 |
|------------|-------|---------------------------------------------|
| `remember` | 4     | Reject the last 4 previously used passwords |

---

### 6. PAM Faillock (system-auth)

**File modified:** `/etc/pam.d/system-auth`

Adds `pam_faillock.so` PAM entries to enforce account lockout at the PAM authentication layer. This is only applied if `pam_faillock.so` is **not already present** in the file.

The following PAM lines are added:

```
auth required pam_faillock.so preauth silent deny=6 unlock_time=300 fail_interval=900
auth [success=1 default=bad] pam_unix.so
auth [default=die] pam_faillock.so authfail deny=6 unlock_time=300 fail_interval=900
account required pam_faillock.so
```

---

### 7. Session Logout (KillUserProcesses)

**File modified:** `/etc/systemd/logind.conf`

Ensures that all user processes are terminated when the user's session ends.

| Parameter            | Value | Description                              |
|----------------------|-------|------------------------------------------|
| `KillUserProcesses`  | yes   | Kill all processes when user logs out    |

> **Service restart:** `systemd-logind` is automatically restarted when this setting changes.

---

### 8. Login Timeout

**File modified:** `/etc/login.defs`

Sets the maximum time allowed for a login attempt before it is terminated.

| Parameter       | Value | Description                              |
|-----------------|-------|------------------------------------------|
| `LOGIN_TIMEOUT` | 30    | Login prompt timeout in seconds          |

---

## Variables Reference

All values are defined in `roles/password_policy/defaults/main.yml` and can be overridden via `--extra-vars`, `group_vars`, or `host_vars`.

| Variable                  | Default | Used In                          |
|---------------------------|---------|----------------------------------|
| `pwquality_minlen`        | 12      | `/etc/security/pwquality.conf`   |
| `pwquality_minclass`      | 4       | `/etc/security/pwquality.conf`   |
| `pwquality_dictcheck`     | 1       | `/etc/security/pwquality.conf`   |
| `pwquality_dcredit`       | -1      | `/etc/security/pwquality.conf`   |
| `pwquality_ucredit`       | -1      | `/etc/security/pwquality.conf`   |
| `pwquality_lcredit`       | -1      | `/etc/security/pwquality.conf`   |
| `pwquality_ocredit`       | -1      | `/etc/security/pwquality.conf`   |
| `pwquality_maxrepeat`     | 1       | `/etc/security/pwquality.conf`   |
| `faillock_deny`           | 6       | `/etc/security/faillock.conf`, PAM |
| `faillock_fail_interval`  | 900     | `/etc/security/faillock.conf`, PAM |
| `faillock_unlock_time`    | 300     | `/etc/security/faillock.conf`, PAM |
| `pass_min_days`           | 1       | `/etc/login.defs`                |
| `pass_max_days`           | 90      | `/etc/login.defs`                |
| `pass_min_len`            | 12      | `/etc/login.defs`                |
| `pass_max_len`            | 12      | `/etc/login.defs`                |
| `login_timeout`           | 30      | `/etc/login.defs`                |
| `password_remember`       | 4       | `/etc/pam.d/system-auth`         |

---

## Usage

### Run the full playbook

```bash
ansible-playbook -i inventory rhel9-password-policy.yml
```

### Dry run (check mode)

```bash
ansible-playbook -i inventory rhel9-password-policy.yml --check --diff
```

### Override variables at runtime

```bash
ansible-playbook -i inventory rhel9-password-policy.yml \
  --extra-vars "pass_max_days=60 faillock_deny=5"
```

---

## Running Specific Sections

Use tags to apply only specific policies:

| Tag                   | Policies Applied                        |
|-----------------------|-----------------------------------------|
| `password_complexity` | Password complexity (pwquality)         |
| `pwquality`           | Password complexity (pwquality)         |
| `faillock`            | Faillock configuration                  |
| `account_lockout`     | Faillock config + PAM faillock entries  |
| `password_age`        | PASS_MIN_DAYS, PASS_MAX_DAYS            |
| `password_length`     | PASS_MIN_LEN, PASS_MAX_LEN             |
| `password_history`    | Password reuse prevention (remember=4)  |
| `session_logout`      | KillUserProcesses setting               |
| `login_timeout`       | LOGIN_TIMEOUT setting                   |

**Example — apply only password aging:**

```bash
ansible-playbook -i inventory rhel9-password-policy.yml --tags password_age
```

---

## Notes

- **Backup files** are created automatically before modifying `/etc/security/pwquality.conf`, `/etc/security/faillock.conf`, `/etc/pam.d/system-auth`, and `/etc/systemd/logind.conf`.
- **Idempotent execution** — running the playbook multiple times produces the same result without duplicating configuration entries.
- **RHEL 9 only** — the pre-task assertion will fail on non-RHEL 9 systems to prevent misconfiguration.
