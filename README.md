# ubuntu_patch

An Ansible role that patches Ubuntu systems within a defined maintenance window and handles reboots automatically.

---

## How it works

The role runs three phases in sequence:

### 1. Window check (`check_window.yml`)

First validates that all four window variables are in `HH:MM` format, that start precedes end for both windows, and that the reboot window falls within the maintenance window. The role fails immediately with a descriptive message if any constraint is violated.

The managed host's local clock is then read and converted to minutes since midnight and compared against the configured maintenance window.

- **Inside the window** — proceeds immediately to patching.
- **Outside the window** — logs how long the wait will be, then pauses Ansible until the window opens. If the window has already passed for the day, the wait extends to the next day's window start. After the wait, time facts are refreshed before the reboot window is evaluated.

> **Note:** `ansible.builtin.pause` halts the Ansible controller process for the calculated duration. This is intentional — it keeps the playbook stateless and avoids needing external schedulers. For very large fleets where hosts span multiple time zones, use separate inventory groups with per-group `group_vars`.

### 2. Patch (`patch.yml`)

First validates that `ubuntu_patch_upgrade_type` is one of the three accepted values; the role fails immediately on an unrecognised value rather than silently skipping all patching.

If `ubuntu_patch_frozen_repo_url` is set, the apt sources file is updated to point at that URL before `apt update` runs. This ensures every host in the fleet installs exactly the same package versions regardless of when the playbook executes. A `.bak` copy of the original sources file is created on first run (`force: false`, so the original is preserved across subsequent runs).

Runs `apt update` then applies updates according to `ubuntu_patch_upgrade_type`:

| Value | Mechanism | Effect |
|---|---|---|
| `security` | `unattended-upgrade -v` | Security updates only (Ubuntu/Debian security origin) |
| `full` | `apt full-upgrade` | All available updates; may install new packages |
| `dist` | `apt dist-upgrade` | All updates; may install or remove packages |

After patching, `/var/run/reboot-required` is checked to determine if a reboot is needed.

### 3. Reboot (`reboot.yml`)

If no reboot is required, logs and exits cleanly. If `ubuntu_patch_reboot_if_required: false`, logs and exits without rebooting.

When a reboot is required and enabled, the host's current time is compared against the reboot window:

```
Maintenance window:   [02:00 ──────────────────────── 06:00]
Reboot window:                        [04:00 ──── 06:00]
                       ^patch here^    ^reboot window^
```

| Current time | Behaviour |
|---|---|
| Before reboot window | Reboot scheduled via `at` to fire at `ubuntu_patch_reboot_window_start`; Ansible does not wait. A flag file at `/var/run/ubuntu_patch_reboot_scheduled` prevents a second `at` job being queued if the playbook runs again before the reboot fires. The flag is cleared automatically on reboot (tmpfs). |
| Inside reboot window | Immediate reboot; Ansible waits for the host to return (up to 600 s) |
| After reboot window | Warning logged; no reboot issued — **manual reboot required** |

---

## Variables

All variables are prefixed `ubuntu_patch_` and can be set in `group_vars`, `host_vars`, or passed via `--extra-vars`.

### Maintenance window

| Variable | Default | Description |
|---|---|---|
| `ubuntu_patch_maintenance_window_start` | `02:00` | Window open time (`HH:MM`, 24-hour, host local time) |
| `ubuntu_patch_maintenance_window_end` | `06:00` | Window close time (`HH:MM`, 24-hour, host local time) |

### Reboot window

The reboot window **must** fall within the maintenance window. The role validates this at startup and fails with a clear message if the constraint is violated.

| Variable | Default | Description |
|---|---|---|
| `ubuntu_patch_reboot_window_start` | `04:00` | Earliest time a reboot may occur |
| `ubuntu_patch_reboot_window_end` | `06:00` | Latest time a reboot may be initiated |
| `ubuntu_patch_reboot_if_required` | `true` | Set to `false` to suppress all reboots |

### Patch behaviour

| Variable | Default | Options | Description |
|---|---|---|---|
| `ubuntu_patch_upgrade_type` | `security` | `security` / `full` / `dist` | Scope of updates to apply |
| `ubuntu_patch_frozen_repo_url` | `""` | Any `http`/`https` URL | Replace all apt source URIs with this URL before patching; empty = use existing mirrors |

---

## Requirements

- Ansible 2.12+
- Target hosts: Ubuntu 18.04 (Bionic), 20.04 (Focal), 22.04 (Jammy), 24.04 (Noble)
- `become: true` (the role requires root to run `apt` and `shutdown`)
- The `at` package is installed automatically when needed for scheduled reboots

---

## Usage

### Minimal inventory

```ini
# inventory/hosts.ini
[webservers]
web01.example.com
web02.example.com

[dbservers]
db01.example.com
```

### group_vars

```yaml
# group_vars/all.yml — applies to every host
ubuntu_patch_maintenance_window_start: "02:00"
ubuntu_patch_maintenance_window_end:   "06:00"
ubuntu_patch_reboot_window_start:      "04:00"
ubuntu_patch_reboot_window_end:        "06:00"
ubuntu_patch_reboot_if_required:       true
ubuntu_patch_upgrade_type:             "security"
```

Override for a specific group — for example, databases that must not reboot automatically:

```yaml
# group_vars/dbservers.yml
ubuntu_patch_reboot_if_required: false
```

### Playbook

```yaml
# site.yml
- name: Patch Ubuntu systems within maintenance window
  hosts: all
  become: true
  roles:
    - ubuntu_patch
```

### Running the playbook

```bash
# Dry run — shows what would change without making any changes
ansible-playbook site.yml --check --diff

# Live run against all hosts
ansible-playbook site.yml

# Target a single group
ansible-playbook site.yml --limit webservers

# Override the upgrade type at run time
ansible-playbook site.yml --extra-vars "ubuntu_patch_upgrade_type=full"

# Pin all hosts to a specific Ubuntu snapshot for reproducible patching
ansible-playbook site.yml --extra-vars "ubuntu_patch_frozen_repo_url=https://snapshot.ubuntu.com/ubuntu/20250501T000000Z"
```

---

## Recommended patterns

### Different windows per environment

Give each environment its own group and `group_vars` directory:

```
inventory/
  hosts.ini
group_vars/
  all.yml              # ubuntu_patch_upgrade_type: security
  production.yml       # stricter window: 03:00–05:00, reboot at 04:30
  staging.yml          # wider window: 01:00–07:00
```

### Rolling updates across a large fleet

Use `serial` to limit how many hosts patch simultaneously, reducing the blast radius if something goes wrong:

```yaml
- name: Patch Ubuntu systems
  hosts: webservers
  become: true
  serial: "25%"          # patch 25 % of hosts at a time
  max_fail_percentage: 10 # abort if more than 10 % of hosts fail
  roles:
    - ubuntu_patch
```

### Scheduled execution via cron

Trigger the playbook just before your maintenance window so hosts that are already in the window begin immediately and early-arriving executions wait:

```cron
# Run at 01:45 — hosts wait up to 15 min then patch from 02:00
45 1 * * 2 ansible-playbook /opt/ansible/ubuntu-patching/site.yml >> /var/log/ansible-patch.log 2>&1
```

### Pinning all hosts to the same package versions (frozen repo)

Set `ubuntu_patch_frozen_repo_url` to a snapshot URL so every host installs identical packages regardless of when the playbook runs. Ubuntu provides an official snapshot service:

```yaml
# group_vars/all.yml
ubuntu_patch_frozen_repo_url: "https://snapshot.ubuntu.com/ubuntu/20250501T000000Z"
```

The role replaces all apt source URIs with this URL before running `apt update`. Both the traditional `/etc/apt/sources.list` format (Ubuntu 22.04 and earlier) and the DEB822 `/etc/apt/sources.list.d/ubuntu.sources` format (Ubuntu 24.04+) are handled automatically.

A `.bak` file is written alongside the original sources file the first time the role runs with a frozen URL. To return to standard mirrors, clear the variable and re-run:

```yaml
ubuntu_patch_frozen_repo_url: ""
```

### Suppress reboots for stateful workloads

```yaml
# host_vars/db01.example.com.yml
ubuntu_patch_reboot_if_required: false
```

Then schedule the reboot separately through your change management process.

---

## Time zone considerations

`ansible_date_time` reflects the **managed host's system clock** in its configured time zone. If hosts are in different time zones, window times are interpreted locally on each host — a `02:00` window means 02:00 wherever that host is. No controller-side conversion is needed; just ensure each host's system time zone is set correctly.

To use UTC windows regardless of host locale, set `TZ=UTC` on the managed hosts or override at the play level:

```yaml
- name: Patch Ubuntu systems
  hosts: all
  become: true
  environment:
    TZ: UTC
  roles:
    - ubuntu_patch
```
