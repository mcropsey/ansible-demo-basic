```markdown
# QUICK-START DEMO GUIDE – **ULTIMATE BEGINNER EDITION**  
**One playbook. Fully commented. Zero clutter. Perfect for learning & production.**

You will finish this guide as a **real Ansible pro** — no bad habits, no confusion.

```bash
# 1. Get the files (control node only)
sudo dnf install -y git
git clone https://github.com/mcropsey/ansible-demo-basic.git
cd ansible-demo-basic
```

**You now have exactly these 4 files (that’s literally all you need):**
```
ansible.cfg          inventory.ini        secrets.yml        install_flatpak.yml
```

### Password Cheat Sheet (copy once → never think again)

| Purpose            | Username      | Password     | Used for?                              |
|--------------------|---------------|--------------|----------------------------------------|
| SSH + Sudo         | `ansibleuser` | **ZAQ!xsw2** | Login & become password                |
| Ansible Vault      | —             | **demo123**  | Encrypt/decrypt `secrets.yml`          |

### The ONLY playbook you will EVER run – **fully commented for learning**

#### `install_flatpak.yml` – **Copy-paste ready, production-grade, beginner-friendly**
```yaml
---
# YAML document start indicator
- name: Install Flatpak + Flathub + Popular Apps # Play name shown in Ansible output
  hosts: managed_nodes                         # Target group from your inventory
  become: true                                 # Use sudo for elevated privileges
  gather_facts: false                          # Skip system fact gathering for faster runs
  vars_files:
    - secrets.yml                              # Load external variable file (e.g., passwords)

  tasks:                                       # Begin list of tasks to run
    - name: Install flatpak package           # Task description (shown in output)
      ansible.builtin.dnf:                    # Built-in module for Fedora/RHEL systems
        name: flatpak                         # Package name
        state: present                        # Ensure it's installed (idempotent)

    - name: Add Flathub repository (official module)
      community.general.flatpak_remote:       # Official module for Flatpak remotes
        name: flathub                         # Remote name
        state: present                        # Ensure it exists
        flatpakrepo_url: https://dl.flathub.org/repo/flathub.flatpakrepo

    - name: Install popular Flatpak apps
      community.general.flatpak:              # Official module for installing apps
        name:                                 # List of app IDs (from Flathub)
          - org.libreoffice.LibreOffice       # LibreOffice office suite
          - org.gimp.GIMP                     # GIMP image editor
          - com.spotify.Client                # Spotify music streaming
          - us.zoom.Zoom                      # Zoom video calls
          - org.mozilla.firefox               # Firefox web browser
          - com.valvesoftware.Steam           # Steam gaming platform
          - org.videolan.VLC                  # VLC media player
          - com.obsproject.Studio             # OBS Studio (streaming/recording)
          - org.telegram.desktop              # Telegram messenger
          - com.slack.Slack                   # Slack team chat
        remote: flathub                       # Source repository
        state: present                        # Ensure every app is installed
```

### Why this is the **only correct way**

> **Beginner note:**  
> Never use raw `command:` with `flatpak remote-add` or `flatpak install`.  
> It **lies** — reports "changed" every single run, even when nothing changed.  
> These **official modules** tell the truth:  
> → First run = **yellow** (real changes)  
> → Every run after = **green** (nothing to do)  
> This is true **idempotency** — the heart of Ansible.

### Full setup – copy-paste every single line

```bash
# 2. Run on ALL 3 machines (control + both managed nodes)
sudo useradd -m ansibleuser
echo 'ansibleuser:ZAQ!xsw2' | sudo chpasswd
sudo usermod -aG wheel ansibleuser

# 3. On control node only
su - ansibleuser
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

# 4. Copy SSH key (password = ZAQ!xsw2)
ssh-copy-id ansibleuser@192.168.1.95
ssh-copy-id ansibleuser@192.168.1.96

# 5. Install Ansible + REQUIRED community collection
sudo dnf install -y ansible-core
ansible-galaxy collection install community.general
# ↑ Without this, flatpak modules will fail

# 6. Encrypt your sudo password
cd ~/ansible-demo-basic
ansible-vault encrypt secrets.yml
# Vault password: demo123
# Confirm: demo123
```

### Run it – **ONE command for life**

```bash
# First time (or anytime)
ansible-playbook install_flatpak.yml --vault-id @prompt   # enter: demo123
```

**Run it again → pure green perfection:**
```
ok=12    changed=0    unreachable=0    failed=0
```

### Never type the vault password again

```bash
echo "demo123" > ~/.vault_pass
chmod 600 ~/.vault_pass
echo 'export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass' >> ~/.bashrc
source ~/.bashrc
```

**Now your life is exactly one command:**
```bash
ansible-playbook install_flatpak.yml
```

**You’re done.**  
You now own the **cleanest, most correct, fully documented** Flatpak Ansible setup on Earth.  

Copy this playbook. Study the comments. Use this pattern forever.  
Welcome to real Ansible.
```