# Role: auto_updates

This role configures `dnf-automatic` on Fedora to automate the downloading and application of system updates.

## Description

The role performs the following actions:
1.  **Installs** the `dnf-automatic` package.
2.  **Deploys** a custom `automatic.conf` to `/etc/dnf/automatic.conf` to enable actual patching (the default OS behavior is download-only).
3.  **Enables and starts** the `dnf-automatic.timer` to ensure updates run on the system schedule.
4.  **Triggers a restart** of the timer via a handler if the configuration is modified.

## Configuration Details

The role replaces the default `/etc/dnf/automatic.conf` with the following optimized settings:

| Setting | Value | Purpose |
| :--- | :--- | :--- |
| `upgrade_type` | `default` | Installs all available updates, including enhancements and bugfixes. |
| `random_sleep` | `0` | Ensures the update starts immediately when the timer triggers, making behavior predictable. |
| `download_updates` | `yes` | Fetches packages to the local cache. |
| `apply_updates` | `yes` | **Critical:** Instructs the system to actually install the updates. |
| `reboot` | `never` | Prevents the machine from rebooting automatically while you are working. |
| `emit_via` | `stdio` | Logs output to the system journal (viewable via `journalctl -u dnf-automatic.timer`). |

## Usage

Add the role to your main playbook (e.g., `local.yml`):

```yaml
- name: Provision Local Machine
  hosts: localhost
  become: true
  roles:
    - auto_updates
```

## Requirements

- **OS**: Fedora
- **Privileges**: Requires `become: true` for package installation and writing to `/etc/`.

## Why this is managed via Ansible

Fedora's default `dnf-automatic` configuration sets `apply_updates = no`. This role ensures that "automatic" means the system is actually patched, rather than just downloading packages and waiting for manual intervention. It also sets `random_sleep` to `0` so that updates don't lag behind the scheduled timer.
