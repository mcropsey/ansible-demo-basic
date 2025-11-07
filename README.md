# Ansible Vault & Variables Quick Reference  
**Demo Build Guide – Fully Organized, Speaker-Ready**  
*Based on: https://github.com/mcropsey/ansible-demo-basic*  
*Target Audience: RHEL-based systems | Beginner to Intermediate Ansible Users*

---

## How to Get the Demo Files from GitHub

> **Talking Points (Before Cloning):**
> - "We’re going to pull a pre-built demo from GitHub. This ensures consistency and saves time."
> - "First, we verify `git` is installed on RHEL — if not, we install it."
> - "Then we clone the repo and explore its structure."

```bash
# Step 1: Verify git is installed (RHEL 8/9)
git --version
# If not found:
sudo dnf install -y git

# Step 2: Clone the demo repo
git clone https://github.com/mcropsey/ansible-demo-basic.git
cd ansible-demo-basic

# Step 3: List files to confirm structure
ls -la
```

**Expected Output After Clone:**
```
ansible.cfg
inventory.ini
secrets.yml
test_vault.yml
install_flatpak.yml
flatpak_demo.yml
README.md
```

---

# DEMO STRUCTURE & TALKING POINTS

---

## Slide 1: Variables in Ansible  
**Docs:** [Ansible Variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html)

> **Talking Points:**
> - Variables reduce repetition and enable dynamic playbooks.
> - **Precedence Order (high to low):**  
>   `command line > role defaults > inventory > playbook > host_vars/group_vars`
> - We’ll use `vars_files` + Vault for secure variable injection.

---

## Slide 2: YAML Syntax & Structure  
**Docs:** [YAML Syntax](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)

> **Talking Points:**
> - YAML is indentation-sensitive (2 spaces, no tabs).
> - Playbooks use dictionaries (`key: value`) and lists (`- item`).
> - `!vault |` is a special tag for encrypted strings.

---

## Slide 3: Ansible Vault CLI Usage  
**Docs:** [ansible-vault](https://docs.ansible.com/ansible/latest/cli/ansible-vault.html)

> **Talking Points:**
> - Vault encrypts sensitive data at rest.
> - Key commands:
>   - `create` → new encrypted file
>   - `edit` / `view` → modify or inspect
>   - `encrypt_string` → inline encryption
>   - `rekey` → change password
> - We use `--vault-id @prompt` for interactive password entry.

---

## Slide 4: Encrypting Variables  
**Docs:** [Encrypting Content](https://docs.ansible.com/ansible/latest/vault_guide/vault_encrypting_content.html)

> **Talking Points:**
> - Never hardcode passwords in plain text.
> - Use `ansible-vault encrypt_string` for one-liners:
>   ```bash
>   ansible-vault encrypt_string 'ZAQ!xsw2' --name 'ansible_become_password'
>   ```
> - Or store in `secrets.yml` via `ansible-vault create`.

---

## Slide 5: Using Vault in Playbooks  
**Docs:** [Vault in Playbooks](https://docs.ansible.com/ansible/latest/vault_guide/vault_using_vault_in_playbooks.html)

> **Talking Points:**
> - Include encrypted files with `vars_files:`
> - Decrypt at runtime using `--vault-id @prompt`
> - Never commit decrypted secrets to Git!

---

## Slide 6: Playbook Basics  
**Tutorial:** [LabEx Playbook Basics](https://labex.io/tutorials/ansible-how-to-use-ansible-playbook-359626)

> **Talking Points:**
> - Minimal playbook needs: `name`, `hosts`, `tasks`
> - `become: true` → sudo
> - `register` → capture command output
> - `changed_when` / `failed_when` → control task status

---

# LAB SETUP (All Nodes)

> **Talking Points (Pre-Demo Prep):**
> - "We need 3 RHEL nodes: 1 control, 2 managed."
> - "All nodes must have `ansibleuser` with sudo access."
> - "SSH key-based auth is required — no password prompts!"

### On **ALL** Machines (Control + Managed)
```bash
# Create ansibleuser
sudo useradd -m ansibleuser
echo 'ansibleuser:ZAQ!xsw2' | sudo chpasswd

# Add to wheel group for sudo
sudo usermod -aG wheel ansibleuser
```

### On **Control Node Only** (as `ansibleuser`)
```bash
su - ansibleuser

# Generate SSH key
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

# Copy to managed nodes
ssh-copy-id ansibleuser@192.168.1.95
ssh-copy-id ansibleuser@192.168.1.96

# Test passwordless SSH
ssh ansibleuser@192.168.1.95 "hostname && whoami"
ssh ansibleuser@192.168.1.96 "hostname && whoami"
```

---

# DEMO 1: Install Ansible & Create Vault

> **Talking Points:**
> - "We install `ansible-core` — lightweight, includes Vault."
> - "Then create `secrets.yml` with the sudo password."

```bash
# Install Ansible
sudo dnf install -y ansible-core

# Create encrypted vault file
ansible-vault create secrets.yml
```

**Paste into `secrets.yml`:**
```yaml
---
# secrets.yml - encrypted with ansible-vault
ansible_become_password: ZAQ!xsw2
```

```bash
# View (prompts for password)
ansible-vault view secrets.yml

# Edit later if needed
ansible-vault edit secrets.yml
```

---

# DEMO 2: Inventory & First Vault Playbook

> **Talking Points:**
> - "Inventory defines targets and connection vars."
> - "`ansible_user` tells Ansible who to SSH as."
> - "Playbook uses vaulted password via `vars_files`."

### `inventory.ini`
```ini
[managed_nodes]
192.168.1.95 ansible_user=ansibleuser
192.168.1.96 ansible_user=ansibleuser
```

### `test_vault.yml`
```yaml
---
- name: Verify sudo password (via Ansible Vault)
  hosts: all
  gather_facts: no
  vars_files:
    - secrets.yml
  vars:
    ansible_become_password: "{{ ansible_become_password }}"

  tasks:
    - name: Verify sudo by becoming root and checking UID
      ansible.builtin.command: id -u
      become: yes
      become_method: sudo
      become_user: root
      register: id_out
      changed_when: false
      failed_when: id_out.stdout != "0"

    - name: Report success
      ansible.builtin.debug:
        msg: "Sudo verified: became root on {{ inventory_hostname }}"
```

**Run It:**
```bash
ansible-playbook -i inventory.ini test_vault.yml --vault-id @prompt
```

> **Expected Output:**
```
TASK [Report success] ************
ok: [192.168.1.95] => { "msg": "Sudo verified: became root on 192.168.1.95" }
ok: [192.168.1.96] => { "msg": "Sudo verified: became root on 192.168.1.96" }
```

---

# DEMO 3: Install Flatpak (Basic – Command Module)

> **Talking Points:**
> - "We’ll install GIMP via Flatpak — two ways."
> - "First: raw `command` module (imperative)."
> - "Use `changed_when: false` to avoid false 'changed' status."

### `install_flatpak.yml`
```yaml
---
- name: Install Flatpak and GIMP on managed nodes
  hosts: managed_nodes
  become: true
  vars_files:
    - secrets.yml
  tasks:
    - name: Ensure Flatpak is installed
      ansible.builtin.dnf:
        name: flatpak
        state: present

    - name: Add Flathub repository if not exists
      ansible.builtin.command:
        cmd: flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
      changed_when: false

    - name: Update Flatpak metadata
      ansible.builtin.command:
        cmd: flatpak update --appstream
      changed_when: false

    - name: Install GIMP from Flathub
      ansible.builtin.command:
        cmd: flatpak install -y flathub org.gimp.GIMP
      changed_when: false
```

**Run:**
```bash
ansible-playbook -i inventory.ini install_flatpak.yml --vault-id @prompt
```

---

# DEMO 4: Best Practice – Use Flatpak Modules

> **Talking Points:**
> - "Ansible has purpose-built modules — idempotent by default."
> - "`community.general` collection adds `flatpak` and `flatpak_remote`."
> - "No need for `changed_when: false` — modules handle state."

### Install Collection
```bash
ansible-galaxy collection install community.general
```

### `flatpak_demo.yml` (Full Tagged Version)
```yaml
---
# ============================================================================
# Flatpak Demo Playbook — Command vs. Flatpak Modules
# Run with tags:
#   --tags basic  → command module approach
#   --tags best   → proper module approach
# ============================================================================

- name: Flatpak Demo (Two Approaches)
  hosts: managed_nodes
  become: true
  gather_facts: false
  vars_files:
    - secrets.yml

  tasks:
    ###########################################################################
    # SECTION 1 — BASIC: COMMAND MODULE
    ###########################################################################
    - name: Ensure Flatpak is installed (Basic)
      ansible.builtin.dnf:
        name: flatpak
        state: present
      tags: basic

    - name: Add Flathub repo (Basic)
      ansible.builtin.command:
        cmd: flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
      changed_when: false
      tags: basic

    - name: Update Flatpak metadata (Basic)
      ansible.builtin.command:
        cmd: flatpak update --appstream -y
      changed_when: false
      tags: basic

    - name: Install GIMP (Basic)
      ansible.builtin.command:
        cmd: flatpak install -y flathub org.gimp.GIMP
      changed_when: false
      tags: basic

    ###########################################################################
    # SECTION 2 — BEST: FLATPAK MODULES
    ###########################################################################
    - name: Ensure Flathub remote exists (Best)
      community.general.flatpak_remote:
        name: flathub
        state: present
        flatpakrepo_url: https://dl.flathub.org/repo/flathub.flatpakrepo
      tags: best

    - name: Ensure GIMP is installed (Best)
      community.general.flatpak:
        name: org.gimp.GIMP
        remote: flathub
        state: present
      tags: best
```

**Run Sections Separately:**
```bash
# Basic (command)
ansible-playbook -i inventory.ini flatpak_demo.yml --tags basic --vault-id @prompt

# Best (modules)
ansible-playbook -i inventory.ini flatpak_demo.yml --tags best --vault-id @prompt
```

---

# OPTIONAL: `ansible.cfg` (Project Local)

> **Talking Points:**
> - "Avoid repeating `become: true` in every playbook."
> - "Local `ansible.cfg` applies only to this directory."

### `ansible.cfg`
```ini
[defaults]
inventory = inventory.ini

[privilege_escalation]
become = true
become_method = sudo
become_user = root
```

---

# FINAL DEMO SUMMARY (Wrap-Up Slide)

| Feature | Demo'd? | Key Takeaway |
|-------|--------|-------------|
| Git Clone | Yes | Always verify tools (`git`) |
| SSH Keys | Yes | Passwordless auth is mandatory |
| Ansible Install | Yes | `ansible-core` is lightweight |
| Vault Create/Edit | Yes | Encrypt sensitive vars |
| Vault in Playbook | Yes | `--vault-id @prompt` |
| Inventory | Yes | `ansible_user` per host |
| `command` Module | Yes | Use `changed_when: false` |
| `community.general` | Yes | Prefer modules over shell |
| Tags | Yes | Run subsets of playbooks |

---

**You’re ready to present!**  
All files are in the GitHub repo.  
Practice with `--vault-id @prompt` and tag filtering.  

> **Pro Tip:** Add `export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass` later for automation (not in demo).
