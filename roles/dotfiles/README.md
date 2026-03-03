# Role: dotfiles

This role manages the deployment of user-specific configuration files (dotfiles) using GNU Stow and initializes development environments for Tmux and Neovim.

## Description

The role automates the following setup:
1.  **Repository Management**: Clones your personal dotfiles repository from GitHub.
2.  **Symlinking**: Uses `stow` to symlink configurations for `kitty`, `nvim`, and `tmux` from the cloned repository to the user's home directory.
3.  **Tmux Setup**: Installs the Tmux Plugin Manager (TPM) and executes a headless plugin installation.
4.  **Neovim Environment**: Creates or updates a specialized Conda environment (`nvim_provider`) using a YAML definition found within the dotfiles repository to ensure Neovim has the necessary Python providers.
5.  **Shell Initialization**: Injects Starship and Conda initialization scripts into the `.bashrc` to ensure the environment is ready upon login.

## Variables

This role relies on variables defined in the main playbook or global scope:
* `target_user`: The username of the account receiving the dotfiles (usually derived from `ansible_env.SUDO_USER`).
* `target_home`: The absolute path to the user's home directory.

## Requirements

* **GNU Stow**: Must be installed on the host (handled in `pre_tasks` of the main playbook).
* **Miniconda**: This role assumes Miniconda is installed at `{{ target_home }}/miniconda3` (provided by the `galaxyproject.miniconda` role).
* **Git**: Required to clone the dotfiles and TPM repositories.

## Usage

Include the role in your playbook after the installation of prerequisite tools:

```yaml
- name: Provision Local Machine
  hosts: localhost
  become: true
  roles:
    - andrewrothstein.starship
    - galaxyproject.miniconda
    - dotfiles
```

## Tasks Overview

1. Clone dotfiles: Fetches the configuration source.
2. Stow configurations: Maps configs to the home directory.
3. Ensure TPM: Clones the Tmux plugin manager.
4. Install Tmux plugins: Runs the TPM installation script headlessly.
5. Conda Environment: Manages the nvim_provider environment to support Neovim plugins.
6. Shell Init: Appends Starship and Conda activation logic to .bashrc.

