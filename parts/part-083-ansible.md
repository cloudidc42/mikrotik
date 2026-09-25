# Part 83: Ansible สำหรับ MikroTik

## สารบัญ
1. [Ansible Setup](#setup)
2. [community.routeros Collection](#collection)
3. [Inventory](#inventory)
4. [Playbooks](#playbooks)
5. [Roles](#roles)
6. [Variables and Templates](#templates)
7. [CI/CD Integration](#cicd)
8. [Lab: Ansible Network Automation](#lab)

---

## 1. Ansible Setup {#setup}

```bash
# ============================================
# ติดตั้ง Ansible และ MikroTik Collection
# ============================================

pip install ansible ansible-pylibssh

# ติดตั้ง community.routeros collection
ansible-galaxy collection install community.routeros

# ตรวจสอบ
ansible --version
ansible-galaxy collection list | grep routeros

# โครงสร้าง project
mkdir -p mikrotik-ansible/{inventory,playbooks,roles,group_vars,host_vars,templates}
cd mikrotik-ansible

# ansible.cfg
cat > ansible.cfg <<'EOF'
[defaults]
inventory = inventory/
roles_path = roles/
host_key_checking = False
timeout = 30
forks = 10
gathering = smart

[persistent_connection]
connect_timeout = 30
command_timeout = 30
EOF
```

---

## 2. community.routeros Collection {#collection}

```yaml
# ============================================
# RouterOS Modules Overview
# ============================================

# community.routeros.command - execute arbitrary commands
# community.routeros.api     - use RouterOS API
# community.routeros.facts   - gather device facts

# Example: gather facts
---
- hosts: mikrotik_routers
  gather_facts: no
  tasks:
    - name: Gather RouterOS facts
      community.routeros.facts:
        gather_subset:
          - hardware
          - config
          - default
      register: facts
    
    - name: Show hostname
      debug:
        msg: "{{ facts.ansible_net_hostname }}"
    
    - name: Show RouterOS version
      debug:
        msg: "{{ facts.ansible_net_version }}"
```

---

## 3. Inventory {#inventory}

```yaml
# inventory/hosts.yml
---
all:
  vars:
    ansible_user: admin
    ansible_connection: network_cli
    ansible_network_os: community.routeros.routeros
    ansible_ssh_pass: "{{ vault_ssh_pass }}"
    ansible_ssh_common_args: '-o StrictHostKeyChecking=no'

mikrotik_core:
  hosts:
    core-01:
      ansible_host: 10.0.0.1
      router_role: core
      location: datacenter
    core-02:
      ansible_host: 10.0.0.2
      router_role: core
      location: datacenter

mikrotik_edge:
  hosts:
    edge-01:
      ansible_host: 10.0.1.1
      router_role: edge
      location: branch-a
    edge-02:
      ansible_host: 10.0.1.2
      router_role: edge
      location: branch-b

mikrotik_all:
  children:
    mikrotik_core:
    mikrotik_edge:
```

```ini
# inventory/hosts.ini
[mikrotik_core]
core-01 ansible_host=10.0.0.1
core-02 ansible_host=10.0.0.2

[mikrotik_edge]
edge-01 ansible_host=10.0.1.1
edge-02 ansible_host=10.0.1.2

[mikrotik_all:children]
mikrotik_core
mikrotik_edge

[mikrotik_all:vars]
ansible_user=admin
ansible_network_os=community.routeros.routeros
ansible_connection=network_cli
```

---

## 4. Playbooks {#playbooks}

```yaml
# playbooks/security_baseline.yml
---
- name: Apply Security Baseline to MikroTik Routers
  hosts: mikrotik_all
  gather_facts: no
  vars:
    management_subnet: "10.0.0.0/24"
    ntp_servers:
      - "203.0.113.100"
      - "203.0.113.101"
  
  tasks:
    - name: Set system name
      community.routeros.command:
        commands:
          - /system identity set name={{ inventory_hostname }}
    
    - name: Disable unused services
      community.routeros.api:
        path: ip service
        handle: ip-service-{{ item.name }}
        data:
          name: "{{ item.name }}"
          disabled: "yes"
      loop:
        - { name: telnet }
        - { name: ftp }
        - { name: www }
      when: item.name in ['telnet', 'ftp', 'www']
    
    - name: Restrict SSH access
      community.routeros.command:
        commands:
          - /ip service set ssh address={{ management_subnet }}
    
    - name: Configure NTP
      community.routeros.command:
        commands:
          - /system ntp client set enabled=yes servers={{ ntp_servers | join(',') }}
    
    - name: Set timezone
      community.routeros.command:
        commands:
          - /system clock set time-zone-name=Asia/Bangkok
    
    - name: Configure syslog
      community.routeros.command:
        commands:
          - /system logging action set remote remote=10.0.0.100 remote-port=514
          - /system logging add topics=firewall,system,account action=remote
    
    - name: Firewall - Allow established
      community.routeros.command:
        commands:
          - >
            /ip firewall filter add chain=input
            connection-state=established,related action=accept
            comment="Allow established (Ansible)"
    
    - name: Firewall - Drop invalid
      community.routeros.command:
        commands:
          - >
            /ip firewall filter add chain=input
            connection-state=invalid action=drop
            comment="Drop invalid (Ansible)"
    
    - name: Save configuration
      community.routeros.command:
        commands:
          - /system backup save name=ansible-backup-{{ ansible_date_time.date }}


# playbooks/vlan_deploy.yml
---
- name: Deploy VLAN Configuration
  hosts: mikrotik_edge
  gather_facts: no
  vars_files:
    - ../group_vars/vlans.yml
  
  tasks:
    - name: Create VLANs
      community.routeros.command:
        commands:
          - >
            /interface vlan add 
            name={{ item.name }}
            vlan-id={{ item.id }}
            interface={{ item.interface }}
            comment="{{ item.comment }}"
      loop: "{{ vlans }}"
    
    - name: Add IP to VLANs
      community.routeros.command:
        commands:
          - >
            /ip address add
            address={{ item.ip }}
            interface={{ item.name }}
      loop: "{{ vlans }}"
      when: item.ip is defined
```

---

## 5. Roles {#roles}

```bash
# สร้าง role structure
ansible-galaxy role init roles/mikrotik_base
ansible-galaxy role init roles/mikrotik_firewall
ansible-galaxy role init roles/mikrotik_bgp
```

```yaml
# roles/mikrotik_base/tasks/main.yml
---
- name: Set system identity
  community.routeros.command:
    commands:
      - /system identity set name={{ inventory_hostname }}
  tags: identity

- name: Set timezone
  community.routeros.command:
    commands:
      - /system clock set time-zone-name={{ timezone | default('UTC') }}
  tags: clock

- name: Configure DNS
  community.routeros.command:
    commands:
      - /ip dns set servers={{ dns_servers | join(',') }} allow-remote-requests={{ dns_allow_remote | default('no') }}
  tags: dns

- name: Enable IP forwarding
  community.routeros.command:
    commands:
      - /ip settings set ip-forward=yes
  tags: forwarding

- name: Update RouterOS
  community.routeros.command:
    commands:
      - /system package update download
  when: update_routeros | default(false)
  tags: update

# roles/mikrotik_base/defaults/main.yml
---
timezone: "Asia/Bangkok"
dns_servers:
  - "8.8.8.8"
  - "8.8.4.4"
dns_allow_remote: "no"
update_routeros: false
```

---

## 6. Variables and Templates {#templates}

```yaml
# group_vars/mikrotik_all.yml
---
ansible_user: admin
ansible_password: "{{ vault_admin_password }}"

ntp_servers:
  - "203.0.113.100"
  - "th.pool.ntp.org"

syslog_server: "10.0.0.50"
management_vlan: 10
management_subnet: "10.0.0.0/24"

default_firewall_rules:
  - chain: input
    state: "established,related"
    action: accept
    comment: "Allow established"
  - chain: input
    state: invalid
    action: drop
    comment: "Drop invalid"

# group_vars/mikrotik_core.yml
---
ospf_area: "0.0.0.0"
ospf_router_id_prefix: "10.0.0."

# host_vars/core-01.yml
---
ospf_router_id: "10.0.0.1"
bgp_as: 65000
bgp_router_id: "10.0.0.1"
```

```jinja2
{# templates/firewall.j2 #}
/ip firewall filter
{% for rule in default_firewall_rules %}
add chain={{ rule.chain }} \
  {% if rule.state is defined %}connection-state={{ rule.state }} \{% endif %}
  action={{ rule.action }} \
  comment="{{ rule.comment }} (Ansible)"
{% endfor %}
```

---

## 7. CI/CD Integration {#cicd}

```yaml
# .github/workflows/network-automation.yml
name: Network Automation CI/CD

on:
  push:
    branches: [main]
    paths:
      - 'mikrotik-ansible/**'
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install ansible ansible-lint yamllint
          ansible-galaxy collection install community.routeros
      
      - name: Lint YAML
        run: yamllint mikrotik-ansible/
      
      - name: Lint Ansible
        run: |
          cd mikrotik-ansible
          ansible-lint playbooks/
      
      - name: Syntax check
        run: |
          cd mikrotik-ansible
          ansible-playbook --syntax-check playbooks/security_baseline.yml -i inventory/hosts.yml

  deploy_staging:
    needs: validate
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: staging
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging routers
        env:
          ANSIBLE_VAULT_PASSWORD: ${{ secrets.VAULT_PASSWORD }}
        run: |
          cd mikrotik-ansible
          echo "$ANSIBLE_VAULT_PASSWORD" > .vault_pass
          ansible-playbook -i inventory/staging.yml \
            playbooks/security_baseline.yml \
            --vault-password-file .vault_pass \
            --limit staging
          rm .vault_pass

  deploy_production:
    needs: deploy_staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to production
        run: |
          cd mikrotik-ansible
          ansible-playbook -i inventory/production.yml \
            playbooks/security_baseline.yml \
            --limit production
```

---

## 8. Lab: Ansible Network Automation {#lab}

### Lab Setup

```bash
# ตัวอย่าง: ตั้งค่า 3 routers ด้วย Ansible

# inventory/lab.yml
all:
  hosts:
    r1: { ansible_host: 192.168.1.1 }
    r2: { ansible_host: 192.168.1.2 }
    r3: { ansible_host: 192.168.1.3 }
  vars:
    ansible_user: admin
    ansible_network_os: community.routeros.routeros
    ansible_connection: network_cli
    ansible_ssh_pass: admin123
```

```bash
# Test connection
ansible -i inventory/lab.yml all -m community.routeros.command \
  -a "commands='/system identity print'"

# Run playbook
ansible-playbook -i inventory/lab.yml playbooks/security_baseline.yml

# Run specific tags
ansible-playbook -i inventory/lab.yml playbooks/security_baseline.yml --tags dns

# Check mode (dry run)
ansible-playbook -i inventory/lab.yml playbooks/security_baseline.yml --check

# Verbose
ansible-playbook -i inventory/lab.yml playbooks/security_baseline.yml -vvv
```

### Verification Checklist

- [ ] Ansible connection ทำงาน
- [ ] Playbook ไม่มี syntax errors
- [ ] Security baseline applied
- [ ] Configuration saved
- [ ] CI/CD pipeline runs
- [ ] Idempotent (run สองครั้ง ผล same)

> **Tip:** ใช้ `--check` mode ก่อน apply จริงเสมอ

> **Note:** community.routeros API module ต้องการ API service enabled บน router

---

## Summary

Part นี้ครอบคลุม:
- **Ansible setup** สำหรับ MikroTik
- **community.routeros** collection
- **Inventory** YAML format
- **Playbooks** สำหรับ security baseline
- **Roles** สำหรับ reusable configuration
- **CI/CD** ด้วย GitHub Actions

---

[← Part 82: Kubernetes](part-082-kubernetes.md) | [Part 84: Terraform →](part-084-terraform.md)
