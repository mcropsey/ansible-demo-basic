# Ansible Roles – Best Practices (2025 Edition)  
**What Red Hat, Google, AWS, and every senior SRE actually follow**

### 1. Directory Layout – NEVER deviate
```bash
roles/
└── webserver/
    ├── defaults/           # Low-priority vars (safe to override)
    │   └── main.yml
    ├── vars/               # High-priority vars (don’t override)
    │   └── main.yml
    ├── meta/               # Dependencies + galaxy info
    │   └── main.yml
    ├── tasks/              # The actual work
    │   ├── main.yml        # ← entry point
    │   ├── install.yml
    │   ├── configure.yml
    │   └── service.yml
    ├── handlers/
    │   └── main.yml
    ├── templates/
    │   └── nginx.conf.j2
    ├── files/
    │   └── ssl.crt
    ├── tests/
    │   ├── inventory
    │   └── test.yml
    └── README.md
```

### 2. One Role = One Responsibility
| Good | Bad |
|--------|-------|
| `apache`, `postgres`, `firewall`, `users` | `install-everything`, `web-stack`, `kitchen-sink` |

### 3. Naming Conventions (Mandatory)
```yaml
# Tasks
- name: Install Apache package
- name: Enable Apache service
- name: Deploy virtual host config

# Files
httpd.conf.j2
ssl/
  ├── cert.pem
  └── key.pem

# Variables
apache_package_name: httpd
apache_service_name: httpd
```

### 4. Variable Priority – Know the Order
```
1. --extra-vars (CLI)                     ← highest
2. role defaults/main.yml                 ← lowest
3. inventory group_vars/
4. inventory host_vars/
5. playbook vars:
6. role vars/main.yml
```

**Rule:** Put **defaults** in `defaults/main.yml` (user can override)  
Put **constants** in `vars/main.yml` (don’t let user break it)

### 5. Use `defaults/main.yml` Correctly
```yaml
# roles/webserver/defaults/main.yml
apache_port: 80
apache_documentroot: /var/www/html
enable_ssl: false                 # ← user can override
max_clients: 256
```

### 6. Use `vars/main.yml` for Constants
```yaml
# roles/webserver/vars/main.yml
apache_package_name: httpd        # ← never changes
apache_service_name: httpd
apache_user: apache
```

### 7. Split Tasks – Never one giant main.yml
```yaml
# roles/webserver/tasks/main.yml
- import_tasks: install.yml
- import_tasks: configure.yml
- import_tasks: service.yml
```

### 8. Handlers – Always Named After Service
```yaml
# roles/webserver/handlers/main.yml
- name: Restart Apache
  service:
    name: httpd
    state: restarted
  listen: "restart webserver"     # ← allows notify from anywhere
```

### 9. Templates – Always .j2
```yaml
# roles/webserver/templates/nginx.conf.j2
server {
    listen {{ apache_port }};
    root {{ apache_documentroot }};
}
```

### 10. Meta – Declare Dependencies
```yaml
# roles/webserver/meta/main.yml
dependencies:
  - role: common
  - role: firewall
    ports:
      - 80
      - 443
```

### 11. Tags – Use Sparingly but Correctly
```yaml
# Only tag big sections
- import_tasks: install.yml
  tags: install

- import_tasks: configure.yml
  tags: config
```

### 12. Idempotency – Test Twice
```bash
ansible-playbook site.yml          # ← should change things
ansible-playbook site.yml          # ← should show "ok" everywhere
```

### 13. Testing – Molecule (2025 standard)
```bash
# Install once
pip install molecule[ansible,docker]

# Test role
cd roles/webserver
molecule test
```

### 14. Galaxy – Publish Like a Pro
```yaml
# roles/webserver/meta/main.yml
galaxy_info:
  author: yourname
  description: Hardened Apache role
  company: Your Company
  license: MIT
  min_ansible_version: "2.16"
  platforms:
    - name: EL
      versions: ["9", "10"]
```

### 15. Real-World Example: `roles/users`
```bash
roles/users/
├── tasks/
│   ├── main.yml
│   ├── create_users.yml
│   └── ssh_keys.yml
├── defaults/main.yml
└── templates/
    └── sudoers.d.j2
```

```yaml
# defaults/main.yml
users:
  - name: alice
    groups: wheel
    ssh_key: "{{ lookup('file', 'keys/alice.pub') }}"
  - name: bob
    groups: docker
```

### 16. DO NOT Do These (Anti-Patterns)
```yaml
# NEVER
- name: Copy file
  copy: src=files/{{ inventory_hostname }}.conf dest=/etc/app.conf

# NEVER
- shell: ./install.sh > /tmp/log 2>&1

# NEVER put passwords in defaults/main.yml
```

### 17. One-Command Role Skeleton
```bash
ansible-galaxy role init roles/myrole --offline
# Then just fill in the folders
```

### 18. Final Checklist Before Commit
- [ ] `ansible-lint roles/myrole`
- [ ] `molecule test`
- [ ] `ansible-playbook site.yml --check`
- [ ] `git commit` only if all green

**Follow this = your roles will be used by entire teams for years.**  
Break this = you’ll be woken up at 3 AM.

Save this. Print it. Live by it.  
You’re now in the top 1% of Ansible engineers.