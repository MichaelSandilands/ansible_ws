# SSH Keys Ansible Role

An idempotent Ansible role for managing SSH keys, SSH config, and Git identity.
Private keys are encrypted with Ansible Vault so they can be safely committed
to a Git repository. The role loops over all keys defined in the `ssh_keys`
variable, so adding a new key is just a matter of adding an entry to the list.

## Directory Structure

```
roles/ssh_keys/
├── defaults/
│   └── main.yml                    # Default variables
├── files/
│   ├── config                      # SSH config file
│   ├── id_ed25519_github           # Vault-encrypted private key (GitHub)
│   ├── id_ed25519_github.pub       # Public key (GitHub)
│   ├── id_ed25519_ec2              # Vault-encrypted private key (EC2)
│   └── id_ed25519_ec2.pub          # Public key (EC2)
└── tasks/
    └── main.yml                    # Role tasks
```

## How the Role Loops Over Keys

The tasks file uses `loop: "{{ ssh_keys }}"` on both the public and private
key deployment tasks. This means the role handles any number of keys without
duplicating tasks. Here is the relevant section from `tasks/main.yml`:

```yaml
- name: Deploy SSH public keys
  ansible.builtin.copy:
    src: "{{ item.public }}"
    dest: "{{ ssh_dir }}/{{ item.public }}"
    owner: "{{ ssh_user }}"
    group: "{{ ssh_user }}"
    mode: "0644"
  loop: "{{ ssh_keys }}"
  loop_control:
    label: "{{ item.public }}"

- name: Deploy SSH private keys (vault-encrypted)
  ansible.builtin.copy:
    src: "{{ item.private }}"
    dest: "{{ ssh_dir }}/{{ item.private }}"
    owner: "{{ ssh_user }}"
    group: "{{ ssh_user }}"
    mode: "0600"
  loop: "{{ ssh_keys }}"
  loop_control:
    label: "{{ item.private }}"
  no_log: true
```

When Ansible encounters a vault-encrypted file in `src`, it automatically
decrypts it in memory and writes the plaintext to `dest`. The `no_log: true`
directive prevents key material from appearing in terminal output.
`loop_control.label` keeps the output clean by only showing the filename
rather than the entire dictionary for each iteration.

## Role Variables

| Variable           | Description                          | Default    |
|--------------------|--------------------------------------|------------|
| `ssh_user`         | Target username                      | detected   |
| `git_user_name`    | Git global user.name                 | `""`       |
| `git_user_email`   | Git global user.email                | `""`       |
| `ssh_keys`         | List of key pairs to deploy          | `[]`       |
| `ssh_config_files` | List of config files to deploy       | `[config]` |
| `git_ssh_signing`  | Optional list of git signing config  | undefined  |

## Usage in Playbook

```yaml
roles:
  - role: ssh_keys
    vars:
      ssh_user: "{{ target_user }}"
      git_user_name: "Michael"
      git_user_email: "michael@example.com"
      ssh_keys:
        - name: id_ed25519_github
          public: id_ed25519_github.pub
          private: id_ed25519_github
        - name: id_ed25519_ec2
          public: id_ed25519_ec2.pub
          private: id_ed25519_ec2
```

## What Gets Deployed

| File                    | Permissions | Notes                         |
|-------------------------|-------------|-------------------------------|
| `~/.ssh/`               | `0700`      | Directory                     |
| `~/.ssh/config`         | `0600`      | SSH host configuration        |
| `~/.ssh/*.pub`          | `0644`      | Public keys (readable)        |
| `~/.ssh/*` (private)    | `0600`      | Private keys (owner-only)     |
| `~/.gitconfig`          | n/a         | Managed via git_config module |

## Running the Playbook

This playbook is designed to be run via `ansible-pull` on a fresh Fedora
installation. The full bootstrap process is:

```bash
# Install ansible
sudo dnf install ansible -y

# Install ansible-galaxy roles
curl -L -o temp_requirements.yml \
  "https://raw.githubusercontent.com/MichaelSandilands/ansible_tuxedo/refs/heads/main/requirements.yml"
ansible-galaxy install -r temp_requirements.yml
rm temp_requirements.yml

# Provision Machine
ansible-pull -U "https://github.com/MichaelSandilands/ansible_tuxedo.git" -K

# Reboot after first provision
# reboot
```

### How This Works

1. **`sudo dnf install ansible -y`** installs Ansible and its dependencies
   from Fedora's official repos.

2. **`curl ... && ansible-galaxy install ...`** downloads `requirements.yml`
   from the repo and installs the third-party Galaxy roles (miniconda, quarto)
   that the playbook depends on. This must happen before `ansible-pull`
   because `ansible-pull` clones into a temporary directory and the Galaxy
   roles need to be pre-installed on the system.

3. **`ansible-pull -U <repo> -K`** does the following:
   - Clones the Git repository into a temporary local checkout.
   - Finds `local.yml` in the repo root (the default playbook name for
     `ansible-pull`).
   - `-K` prompts for the sudo password (needed for `become: true` tasks
     like dnf installs and systemd configuration).
   - The `ansible.cfg` in the repo root contains `ask_vault_pass = True`,
     which causes Ansible to also prompt for the vault password before
     execution begins.
   - Ansible then runs the playbook locally against `localhost`.

### How Vault Decryption Works

The private key files in `roles/ssh_keys/files/` are encrypted with Ansible
Vault. They look like this in the repo:

```
$ANSIBLE_VAULT;1.1;AES256
38616538653134653962643237613866616161383437663463356265343262633031333536643739
...
```

When the playbook runs, this is the decryption flow:

1. `ansible-pull` starts and reads `ansible.cfg` from the repo, which sets
   `ask_vault_pass = True`.
2. Ansible prompts: `Vault password:` — you enter the password you used
   when you originally ran `ansible-vault encrypt`.
3. Ansible derives an AES-256 decryption key from your password.
4. When the `copy` task for a private key executes, Ansible detects the
   `$ANSIBLE_VAULT` header in the source file, decrypts it in memory using
   the derived key, and writes the plaintext private key to `~/.ssh/` with
   `0600` permissions.
5. The decrypted key never appears in Ansible's terminal output because the
   task has `no_log: true` set.
6. This happens once per iteration of the loop — so for two keys (GitHub
   and EC2), the copy task runs twice, decrypting each key individually.

The encrypted files in the Git repo remain encrypted. Only the deployed copies
in `~/.ssh/` are plaintext, and they are protected by filesystem permissions.

## Ansible Vault Password

You need to remember the vault password you used when encrypting your keys.
There is no password file committed to the repo — the `ansible.cfg` setting
`ask_vault_pass = True` ensures you are prompted interactively every time.

If you are developing locally and want to avoid repeated prompts, you can
create a local password file (never commit this):

```bash
echo 'your-vault-password' > .vault_pass
chmod 600 .vault_pass
```

Then run with:

```bash
ansible-playbook local.yml --vault-password-file .vault_pass -K
```

Make sure `.vault_pass` is in your `.gitignore`.

## Adding a New SSH Key (Full Workflow)

This example adds an EC2 key alongside the existing GitHub key. The same
process applies for any service — GitLab, Bitbucket, another server, etc.

### Step 1: Generate the Key Pair

```bash
ssh-keygen -t ed25519 -C "michael@example.com" -f ~/.ssh/id_ed25519_ec2
```

This creates two files:

- `~/.ssh/id_ed25519_ec2` (private key)
- `~/.ssh/id_ed25519_ec2.pub` (public key)

For EC2 specifically: if AWS generated a `.pem` key pair for you when
launching the instance, you can use that instead of generating a new one.
Skip to Step 3 and encrypt the `.pem` file as the private key.

### Step 2: Add the Public Key to Your Service

For **EC2**, append the public key to `~/.ssh/authorized_keys` on the
instance. If you still have access via the AWS-generated key:

```bash
ssh -i ~/.ssh/aws-original.pem ec2-user@your-ec2-host \
  "echo '$(cat ~/.ssh/id_ed25519_ec2.pub)' >> ~/.ssh/authorized_keys"
```

For **GitHub**: Settings → SSH and GPG keys → New SSH key.

For **GitLab**: Preferences → SSH Keys → Add new key.

### Step 3: Copy the Public Key into the Role

```bash
cp ~/.ssh/id_ed25519_ec2.pub roles/ssh_keys/files/
```

Public keys are safe to commit in plaintext.

### Step 4: Encrypt the Private Key with Ansible Vault

```bash
ansible-vault encrypt ~/.ssh/id_ed25519_ec2 \
  --output roles/ssh_keys/files/id_ed25519_ec2
```

Ansible will prompt for a vault password. Use the same password you use for
all other vault-encrypted files in this repo.

To verify it was encrypted correctly:

```bash
head -1 roles/ssh_keys/files/id_ed25519_ec2
# Should output: $ANSIBLE_VAULT;1.1;AES256
```

### Step 5: Update the SSH Config

Edit `roles/ssh_keys/files/config` and add a host block for the new key:

```
# GitHub
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github
    IdentitiesOnly yes

# EC2 Server
Host ec2
    HostName your-ec2-ip-or-hostname
    User ec2-user
    IdentityFile ~/.ssh/id_ed25519_ec2
    IdentitiesOnly yes

# Default settings
Host *
    AddKeysToAgent yes
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

The `Host ec2` alias lets you connect with just `ssh ec2` instead of
typing out the full hostname and user every time. Replace
`your-ec2-ip-or-hostname` with your instance's public IP or DNS name.
Change `ec2-user` to `ubuntu` if you're running Ubuntu on the instance.

### Step 6: Add the Key to `ssh_keys` in Your Playbook

```yaml
ssh_keys:
  - name: id_ed25519_github
    public: id_ed25519_github.pub
    private: id_ed25519_github
  - name: id_ed25519_ec2
    public: id_ed25519_ec2.pub
    private: id_ed25519_ec2
```

The loop in `tasks/main.yml` picks up the new entry automatically. No task
changes are needed.

### Step 7: Commit the Encrypted Key

```bash
git add roles/ssh_keys/files/id_ed25519_ec2
git add roles/ssh_keys/files/id_ed25519_ec2.pub
git add roles/ssh_keys/files/config
git commit -m "Add EC2 SSH key"
git push
```

The private key in the repo is AES-256 encrypted. Without the vault password,
it is unreadable ciphertext.

### Step 8: Provision and Test

On a fresh machine (or to update an existing one):

```bash
ansible-pull -U "https://github.com/MichaelSandilands/ansible_tuxedo.git" -K
```

Enter your sudo password when prompted, then your vault password. After the
playbook completes:

```bash
# Test GitHub
ssh -T git@github.com
# Hi michael! You've successfully authenticated...

# Test EC2
ssh ec2
# Welcome to Amazon Linux...
```

## Managing Existing Keys

### View an Encrypted Key (without decrypting to disk)

```bash
ansible-vault view roles/ssh_keys/files/id_ed25519_ec2
```

### Re-encrypt All Keys with a New Vault Password

```bash
ansible-vault rekey roles/ssh_keys/files/id_ed25519_github
ansible-vault rekey roles/ssh_keys/files/id_ed25519_ec2
```

### Decrypt a Key Temporarily (e.g. for manual use)

```bash
ansible-vault decrypt roles/ssh_keys/files/id_ed25519_ec2 \
  --output /tmp/my_key
chmod 600 /tmp/my_key
# Use the key, then remove it
rm /tmp/my_key
```

## Git Identity and .gitconfig

The role sets `user.name`, `user.email`, and `init.defaultBranch` using the
`git_config` module. This writes to `~/.gitconfig` via git itself, so it
merges with any existing settings rather than overwriting the file.

There is no need to encrypt `.gitconfig`. Your name and email appear in
`git log` on every public repo you contribute to — they are public by design.

To configure SSH commit signing, add the optional variable:

```yaml
git_ssh_signing:
  - key: gpg.format
    value: ssh
  - key: user.signingKey
    value: "~/.ssh/id_ed25519_github.pub"
  - key: commit.gpgSign
    value: "true"
  - key: tag.gpgSign
    value: "true"
```

## Security Notes

- **Never commit unencrypted private keys.** Always run
  `ansible-vault encrypt` before copying into `files/`.
- **Never commit `.vault_pass`.** Add it to `.gitignore` if you use one
  locally.
- **Vault-encrypted files** use AES-256 encryption. They are safe to store
  in a public Git repository.
- **Public keys** are safe to commit in plaintext. They are public by design.
- **`no_log: true`** is set on the private key deployment task to prevent
  Ansible from printing key contents to the terminal.
- **All keys share one vault password.** Use the same password when encrypting
  every private key so that a single prompt at runtime decrypts them all.
- If you rotate your vault password, use `ansible-vault rekey` on every
  encrypted file in `files/`.

## .gitignore

Make sure your repo's `.gitignore` includes:

```
.vault_pass
*.retry
```
