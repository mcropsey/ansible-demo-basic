# QUICK-START DEMO GUIDE


```bash
# 1. Get my files (control node only)
sudo dnf install -y git
git clone https://github.com/mcropsey/ansible-demo-basic.git
cd ansible-demo-basic
```

**You now have these exact files:**
```
ansible.cfg          inventory.ini        secrets.yml
test_vault.yml       install_flatpak.yml  flatpak_demo.yml
```

### File contents (exactly as in repo)

#### `inventory.ini`
```ini
[managed_nodes]
192.168.1.95 ansible_user=ansibleuser
192.168.1.96 ansible_user=ansibleuser
```

#### `ansible.cfg`
```ini
[defaults]
inventory = inventory.ini

[privilege_escalation]
become = true
become_method = sudo
become_user = root
```

#### `secrets.yml` (plain text – will encrypt)
```yaml
---
ansible_become_password: ZAQ!xsw2
```

#### `test_vault.yml`
```yaml
---
- name: Test Vault – Verify sudo works
  hosts: all
  gather_facts: no
  vars_files:
    - secrets.yml

  tasks:
    - name: Become root and check UID
      ansible.builtin.command: id -u
      become: yes
      register: id_out
      changed_when: false
      failed_when: id_out.stdout != "0"

    - name: Success
      ansible.builtin.debug:
        msg: "Vault works! Root on {{ inventory_hostname }}"
```

#### `flatpak_demo.yml`
```yaml
---
- name: Flatpak Demo – Command vs Modules
  hosts: managed_nodes
  become: true
  gather_facts: false
  vars_files:
    - secrets.yml

  tasks:
    - name: Install flatpak package
      ansible.builtin.dnf:
        name: flatpak
        state: present
      tags: basic

    - name: Add Flathub (command)
      ansible.builtin.command: flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
      changed_when: false
      tags: basic

    - name: Install GIMP (command)
      ansible.builtin.command: flatpak install -y flathub org.gimp.GIMP
      changed_when: false
      tags: basic

    - name: Add Flathub remote (module)
      community.general.flatpak_remote:
        name: flathub
        state: present
        flatpakrepo_url: https://dl.flathub.org/repo/flathub.flatpakrepo
      tags: best

    - name: Install GIMP (module)
      community.general.flatpak:
        name: org.gimp.GIMP
        remote: flathub
        state: present
      tags: best
```

### Setup (run exactly in order)

```bash
# 2. On ALL 3 machines (control + both managed)
sudo useradd -m ansibleuser
echo 'ansibleuser:ZAQ!xsw2' | sudo chpasswd
sudo usermod -aG wheel ansibleuser

# 3. On control node only
su - ansibleuser
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

# 4. Copy keys to managed nodes
ssh-copy-id ansibleuser@192.168.1.95
ssh-copy-id ansibleuser@192.168.1.96
# password: ZAQ!xsw2 (once)

# 5. Install Ansible + collection
sudo dnf install -y ansible-core
ansible-galaxy collection install community.general

# 6. Encrypt secrets (ONE COMMAND)
cd ~/ansible-demo-basic
ansible-vault encrypt secrets.yml
# Vault password: demo123
# Confirm: demo123
```

### Run demos (copy-paste)

```bash
# Test vault
ansible-playbook test_vault.yml --vault-id @prompt          # enter: demo123

# Flatpak – command way
ansible-playbook flatpak_demo.yml --tags basic --vault-id @prompt

# Flatpak – best way
ansible-playbook flatpak_demo.yml --tags best --vault-id @prompt
```

### Optional: No more typing vault password

```bash
echo "demo123" > ~/.vault_pass
chmod 600 ~/.vault_pass
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass

# Now just:
ansible-playbook flatpak_demo.yml --tags best
```

