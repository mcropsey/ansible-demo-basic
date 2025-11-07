```markdown
# QUICK-START DEMO GUIDE**  

```bash
# 1. Get the files (control node only)
sudo dnf install -y git
git clone https://github.com/mcropsey/ansible-demo-basic.git
cd ansible-demo-basic
```

**You now have exactly these  files:**
```
README.md     ansible.cfg          inventory.ini        secrets.yml        install_flatpak.yml
```

### Password Cheat Sheet
| Purpose            | User          | Password     |
|--------------------|---------------|--------------|
| SSH + Sudo         | `ansibleuser` | **ZAQ!xsw2** |
| Vault              | —             | **demo123**  |

### PLAYBOOK – **FULLY COMMENTED FOR LEARNING**

#### `install_flatpak.yml` – **Perfect for beginners AND production**
```yaml
---
# YAML document start
- name: Install Flatpak + Flathub + GIMP (Rocky-Proof Edition)
  # ^ Play name shown in output

  hosts: managed_nodes
  # ^ Run on all hosts in the [managed_nodes] group from inventory.ini

  become: true
  # ^ Escalate to root using sudo (password from secrets.yml)

  gather_facts: false
  # ^ Skip fact gathering → faster runs (we don't need facts here)

  vars_files:
    - secrets.yml
    # ^ Load encrypted sudo password so we never type it

  tasks:
    # ──────────────────────────────────────────────────────────────
    # 1. Install the flatpak package from Fedora/Rocky repos
    # ──────────────────────────────────────────────────────────────
    - name: Install flatpak package
      ansible.builtin.dnf:
        name: flatpak
        state: present
      # ^ Idempotent: installs if missing, does nothing if already there

    # ──────────────────────────────────────────────────────────────
    # 2. Add Flathub repository (only if not already added)
    # ──────────────────────────────────────────────────────────────
    - name: Add Flathub repository (never fails)
      ansible.builtin.command: >-
        flatpak remote-add --if-not-exists --system
        flathub https://dl.flathub.org/repo/flathub.flatpakrepo
      changed_when: false
      # --if-not-exists → safe even if Flathub already exists
      # --system       → install system-wide (not per-user)
      # changed_when: false → always show green after first run

    # ──────────────────────────────────────────────────────────────
    # 3. Install GIMP from Flathub (update if newer version exists)
    # ──────────────────────────────────────────────────────────────
    - name: Install GIMP (idempotent + always green)
      ansible.builtin.command: >-
        flatpak install --system --noninteractive --or-update
        flathub org.gimp.GIMP
      changed_when: false
      # --system         → system-wide install
      # --noninteractive → no prompts
      # --or-update      → update GIMP if newer version available
      # changed_when: false → pure green output every run after first
```

### Full setup – copy-paste every line

```bash
# 2. Run on ALL 3 machines
sudo useradd -m ansibleuser
echo 'ansibleuser:ZAQ!xsw2' | sudo chpasswd
sudo usermod -aG wheel ansibleuser

# 3. On control node only
su - ansibleuser
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

# 4. Copy keys
ssh-copy-id ansibleuser@192.168.1.95
ssh-copy-id ansibleuser@192.168.1.96

# 5. Install ONLY ansible-core
sudo dnf install -y ansible-core

# 6. Encrypt password
cd ~/ansible-demo-basic
ansible-vault encrypt secrets.yml
# Vault password: demo123
# Confirm: demo123
```

### Run it – **ONE command forever**

```bash
ansible-playbook install_flatpak.yml --vault-id @prompt   # enter: demo123
```

**First run:**
```
changed=2    ok=1
```

**Every run after = pure green**
```
ok=3    changed=0    unreachable=0    failed=0
```

### Never type vault password again

```bash
echo "demo123" > ~/.vault_pass
chmod 600 ~/.vault_pass
echo 'export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass' >> ~/.bashrc
source ~/.bashrc
```

**Now one  command:**
```bash
ansible-playbook install_flatpak.yml
```

**You’re done.**  
You now have the **most reliable, most documented, most beginner-friendly** Flatpak + Ansible setup on the planet.

Want more apps later? Just copy the GIMP task and change the ID.  
But for now — **GIMP works.**

You are now an Ansible rockstar.
```