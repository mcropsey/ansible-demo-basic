# **DEMO: Ansible Vault & Variables Quick Reference**  
**Presenter:** *You* | **Audience:** DevOps Beginners / Security-Conscious Admins  
**Goal:** Show **secure variable handling** with `ansible-vault`, **playbook structure**, and **best practices** using real-world Flatpak + GIMP install.

---

## **SLIDE 1: Agenda & Objectives**  
### **Talking Points (30 sec)**  
> “Today we’ll cover **Ansible variables**, **Vault encryption**, and **two ways to install Flatpak apps** — one basic with `command`, one *best practice* with proper modules.  
> By the end, you’ll know how to:  
> - Encrypt secrets safely  
> - Use them in playbooks  
> - Avoid anti-patterns like `command` + `changed_when: false`  
> Let’s jump in!”

---

## **SLIDE 2: Variables in Ansible**  
[Link: Ansible Docs – Using Variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html)

### **Talking Points (1 min)**  
> “Variables are the backbone of Ansible.  
> - Defined in: inventory, playbooks, `host_vars/`, `group_vars/`, roles, etc.  
> - **Precedence matters** — command line > playbook > inventory  
> - Scoping: `host`, `group`, `play`, `task`  
>  
> **Security risk:** Plaintext passwords in files = bad.  
> → This is why **Ansible Vault** exists.”

---

## **SLIDE 3: YAML Syntax & Structure**  
[Link: Ansible Docs – YAML Syntax](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)

### **Talking Points (45 sec)**  
> “Ansible uses YAML — indentation = structure.  
> - `key: value` → dictionary  
> - `- item` → list  
> - `!vault |` → encrypted block  
>  
> **Pro tip:** Use `yamllint` or VS Code to catch syntax errors early.”

---

## **SLIDE 4: ansible-vault CLI Reference**  
[Link: ansible-vault CLI](https://docs.ansible.com/ansible/latest/cli/ansible-vault.html)

### **Talking Points (1 min)**  
> “Vault is your **password manager for Ansible**.  
> Key commands:  
> - `create` → new encrypted file  
> - `edit` / `view` → modify safely  
> - `encrypt` / `decrypt` / `rekey`  
> - `encrypt_string` → inline secrets  
>  
> We’ll use `--vault-id @prompt` for interactive demos — **no passwords in scripts**.”

---

## **SLIDE 5: Environment Setup (Live or Pre-Done)**  
### **Talking Points (1 min)**  
> “Let’s set the stage:  
> 1. **All boxes:** `ansibleuser` + password `ZAQ!xsw2`, member of `wheel`  
> 2. **Control node:**  
>    - SSH key generated + copied  
>    - `ansible-core` installed via `dnf`  
> 3. **Passwordless SSH** verified with `df -h && hostname`  
>  
> → Now we can run playbooks **without prompts** (except Vault)”

**Live Demo Snippet (Optional):**  
```bash
ssh ansibleuser@192.168.1.95 "df -h && hostname"
```

---

## **SLIDE 6: Creating the Vault File**  
### **Talking Points (1 min)**  
> “We store **only** the sudo password in Vault — nothing else.  
> File: `secrets.yml`”

**Live Command:**  
```bash
ansible-vault create secrets.yml
```

**Paste this content (when prompted):**
```yaml
---
# secrets.yml - encrypted with ansible-vault
ansible_become_password: ZAQ!xsw2
```

**Verify:**  
```bash
ansible-vault view secrets.yml --vault-id @prompt
```

> “Vault files are **AES-256 encrypted**. Never commit plaintext!”

---

## **SLIDE 7: Inventory File**  
### **Talking Points (30 sec)**  
> “Simple INI inventory — no secrets here.”

**File: `inventory.ini`**
```ini
[managed_nodes]
192.168.1.95 ansible_user=ansibleuser
192.168.1.96 ansible_user=ansibleuser
```

> “We’ll reference this with `-i inventory.ini`”

---

## **SLIDE 8: First Playbook — Vault Test**  
**File: `test_vault.yml`**

### **Talking Points (1 min)**  
> “Goal: Prove we can **become root** using the encrypted password.”

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

**Run:**  
```bash
ansible-playbook -i inventory.ini test_vault.yml --vault-id @prompt
```

**Expected Output:**  
```
ok: [192.168.1.95] => { "msg": "Sudo verified: became root on 192.168.1.95" }
```

> “Vault password entered once → decrypted at runtime → never logged.”

---

## **SLIDE 9: install_flatpak.yml — Basic Approach (command module)**  
**File: `install_flatpak.yml`**

### **Talking Points (1 min)**  
> “This works… but it’s **not idempotent** and **hard to maintain**.”

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

> “Notice `changed_when: false` → lies to Ansible.  
> Not ideal for production.”

---

## **SLIDE 10: Best Practice — Install community.general**  
### **Talking Points (45 sec)**  
> “Ansible core doesn’t include Flatpak modules.  
> We install the **community collection** from Galaxy.”

**Run:**  
```bash
ansible-galaxy collection install community.general
```

> “Now we get:  
> - `community.general.flatpak`  
> - `community.general.flatpak_remote`  
> → **Idempotent, clean, readable**”

---

## **SLIDE 11: flatpak_demo.yml — Two Approaches Side-by-Side**  
**File: `flatpak_demo.yml`**

### **Talking Points (2 min)**  
> “Same goal, two philosophies:  
> - `--tags basic` → `command` module (anti-pattern)  
> - `--tags best` → proper modules (recommended)”

```yaml
---
# ============================================================================
# Flatpak Demo Playbook — Command vs. Flatpak Modules
# Run: ansible-playbook flatpak_demo.yml --tags basic
#      ansible-playbook flatpak_demo.yml --tags best
# ============================================================================

- name: Flatpak Demo (Two Approaches)
  hosts: managed_nodes
  become: true
  gather_facts: false

  tasks:
    ###########################################################################
    # SECTION 1 — BASIC APPROACH USING COMMAND MODULE
    ###########################################################################
    - name: Ensure Flatpak is installed (Basic)
      ansible.builtin.dnf:
        name: flatpak
        state: present
      tags: [basic]

    - name: Add Flathub repo (Basic)
      ansible.builtin.command:
        cmd: flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
      changed_when: false
      tags: [basic]

    - name: Update Flatpak metadata (Basic)
      ansible.builtin.command:
        cmd: flatpak update --appstream -y
      changed_when: false
      tags: [basic]

    - name: Install GIMP using command module (Basic)
      ansible.builtin.command:
        cmd: flatpak install -y flathub org.gimp.GIMP
      changed_when: false
      tags: [basic]

    ###########################################################################
    # SECTION 2 — BEST PRACTICE USING FLATPAK MODULES
    ###########################################################################
    - name: Ensure Flathub remote exists (Best)
      community.general.flatpak_remote:
        name: flathub
        state: present
        flatpakrepo_url: https://dl.flathub.org/repo/flathub.flatpakrepo
      tags: [best]

    - name: Ensure GIMP is installed from Flathub (Best)
      community.general.flatpak:
        name: org.gimp.GIMP
        remote: flathub
        state: present
      tags: [best]
```

---

## **SLIDE 12: Run the Demos**  
### **Talking Points (2 min)**  

#### **1. Basic Approach**  
```bash
ansible-playbook -i inventory.ini flatpak_demo.yml --tags basic --vault-id @prompt
```

> “Works, but **always ‘changed’** or **lies with `changed_when`**”

#### **2. Best Practice**  
```bash
ansible-playbook -i inventory.ini flatpak_demo.yml --tags best --vault-id @prompt
```

> “**Idempotent** — runs once, skips on rerun.  
> Clean output. No hacks.”

---

## **SLIDE 13: Optional: ansible.cfg**  
**File: `ansible.cfg` (in project dir)**

### **Talking Points (30 sec)**  
> “Avoid repeating `become: true` in every play.”

```ini
[privilege_escalation]
become=True
become_method=sudo
become_user=root
```

> “Now all playbooks in this dir default to sudo.”

---

## **SLIDE 14: Key Takeaways**  
### **Talking Points (1 min)**  

| Do | Don't |
|------|---------|
| Use `ansible-vault` for secrets | Store passwords in plaintext |
| Use `--vault-id @prompt` in demos | Hardcode vault passwords |
| Prefer modules over `command` | Use `changed_when: false` to fake idempotency |
| Use `ansible-galaxy collection install` | Reinvent the wheel |

> “**Secure + Idempotent + Maintainable** = Production-ready Ansible”

---

## **SLIDE 15: Resources & Next Steps**  
### **Talking Points (30 sec)**  
- [Ansible Vault Docs](https://docs.ansible.com/ansible/latest/user_guide/vault.html)  
- [community.general Collection](https://galaxy.ansible.com/community/general)  
- Try: Encrypt a DB password, use in `postgresql_user` module  
- Challenge: Write a role with encrypted `defaults/main.yml`

---

## **Q&A + Wrap-Up**  
> “Any questions?  
> Let’s encrypt something live if time allows!”

---

# **Downloadable Demo Bundle**  
Create a zip with:
```
demo/
├── inventory.ini
├── secrets.yml (encrypted)
├── test_vault.yml
├── install_flatpak.yml
├── flatpak_demo.yml
├── ansible.cfg
└── README.md (this guide)
```

---

