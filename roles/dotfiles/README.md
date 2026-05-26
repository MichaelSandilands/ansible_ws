# Role: dotfiles

This role manages the deployment of user-specific configuration files (dotfiles) using GNU Stow and initializes development environments for Tmux and Neovim.

## Description

The role automates the following setup:
1.  **Repository Management**: Clones your personal dotfiles repository from GitHub.
2.  **Symlinking**: Uses `stow` to symlink configurations for `kitty`, `nvim`, and `tmux` from the cloned repository to the user's home directory.
3.  **Tmux Setup**: Installs the Tmux Plugin Manager (TPM) and executes a headless plugin installation.
4.  **Neovim Environment**: Creates a dedicated Python virtual environment at `~/.virtualenvs/neovim` and installs the remote-plugin dependencies (pynvim, jupyter_client, and Molten's optional rendering packages). Neovim's `python3_host_prog` points at this venv.
5.  **Shell Initialization**: Injects the Starship prompt initialization into `.bashrc` so the prompt is ready upon login.

## Variables

This role relies on variables defined in the main playbook or global scope:
* `target_user`: The username of the account receiving the dotfiles (usually derived from `ansible_env.SUDO_USER`).
* `target_home`: The absolute path to the user's home directory.

## Requirements

* **GNU Stow**: Must be installed on the host (handled in `pre_tasks` of the main playbook).
* **Python 3**: Must be present with the `venv` module (ships with Fedora's `python3`). Used to build the `~/.virtualenvs/neovim` provider environment.
* **Git**: Required to clone the dotfiles and TPM repositories.

## Usage

Include the role in your playbook after the installation of prerequisite tools:

```yaml
- name: Provision Local Machine
  hosts: localhost
  become: true
  roles:
    - andrewrothstein.starship
    - dotfiles
```

## Tasks Overview

1. Clone dotfiles: Fetches the configuration source.
2. Stow configurations: Maps configs to the home directory.
3. Ensure TPM: Clones the Tmux plugin manager.
4. Install Tmux plugins: Runs the TPM installation script headlessly.
5. Neovim provider venv: Creates `~/.virtualenvs/neovim` and installs the Python remote-plugin dependencies.
6. Shell Init: Appends the Starship initialization to .bashrc.

