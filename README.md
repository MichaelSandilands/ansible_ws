# Fedora Workstation Provisioning

This repository contains Ansible playbooks and roles to automate the setup of a Fedora workstation. It configures system repositories, installs development tools (Docker, Nvidia, R, Miniconda), manages SSH keys, and deploys dotfiles using GNU Stow.

## Quick Start

Run the following commands to provision your local machine:

```bash
sudo dnf upgrade -y
# 1. Install Ansible
sudo dnf install ansible git -y

# 3. Provision Machine
# Pulls this repository and executes the local.yml playbook
ansible-pull -U "https://github.com/MichaelSandilands/ansible_ws.git" -K
```

Note: You will be prompted for your sudo password (-K) and the Ansible Vault password (configured in ansible.cfg) to decrypt sensitive data like SSH keys.

## Command Explanations

`ansible-pull`

Used for "pull-mode" deployment where the target machine pulls the configuration from Git and applies it to itself.

- `-U`: The URL of this repository.

- `-K`: (Ask-become-pass) Prompts for your sudo password so Ansible can install system packages.

## Post-Installation Notes

1. Reboot

It is highly recommended to reboot after the first successful provision, especially if kernel updates, Nvidia drivers, or Docker were installed.

2. Neovim & Molten Issues

See my [dotfiles repo](https://github.com/MichaelSandilands/dotfiles.git)

## Role Overview

- **auto_updates**: Configures `dnf5-automatic` for hands-off system patching.
- **dotfiles**: Clones your dotfiles repo, uses `stow` for symlinking, and initializes the Starship prompt.
- **ssh_keys**: Securely deploys SSH private/public keys and configurations from encrypted vault files.
