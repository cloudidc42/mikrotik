# Part 88: CI/CD สำหรับ Network Configuration

## สารบัญ
1. [GitOps for Network](#gitops)
2. [Config Validation](#validation)
3. [Pipeline Design](#pipeline)
4. [Automated Testing](#testing)
5. [Rollback Strategy](#rollback)
6. [Lab: Network GitOps](#lab)

---

## 1. GitOps for Network {#gitops}

```
Developer → Git Commit → CI/CD Pipeline
                              │
                    ┌─────────┴─────────┐
                    │                   │
               [Validate]          [Test Lab]
                    │                   │
                    └─────────┬─────────┘
                              │ Pass
                    ┌─────────▼─────────┐
                    │  Deploy Staging   │
                    └─────────┬─────────┘
                              │ Manual Approval
                    ┌─────────▼─────────┐
                    │ Deploy Production │
                    └───────────────────┘
```

### Repository Structure

```
network-configs/
├── inventory/
│   ├── production.yml
│   └── staging.yml
├── host_vars/
│   ├── core-01.yml
│   └── edge-01.yml
├── group_vars/
│   └── mikrotik_all.yml
├── templates/
│   ├── firewall.j2
│   └── bgp_peers.j2
├── playbooks/
│   ├── deploy.yml
│   └── validate.yml
├── tests/
│   ├── test_connectivity.py
│   └── test_services.py
├── scripts/
│   └── validate_config.py
└── .github/
    └── workflows/
        └── network-cicd.yml
```

---

## 2. Config Validation {#validation}

```python
# scripts/validate_config.py

import yaml
import re
import sys
from pathlib import Path
from typing import List, Dict, Tuple

class MikroTikConfigValidator:
    """Validate network configuration files ก่อน deploy"""
    
    def __init__(self):
        self.errors: List[str] = []
        self.warnings: List[str] = []
    
    def validate_ip_address(self, ip: str) -> bool:
        """Validate IP/CIDR format"""
        pattern = r'^(\d{1,3}\.){3}\d{1,3}(\/\d{1,2})?$'
        if not re.match(pattern, ip):
            return False
        
        octets = ip.split('/')[0].split('.')
        return all(0 <= int(o) <= 255 for o in octets)
    
    def validate_vlan_id(self, vlan_id: int) -> bool:
        """Validate VLAN ID (1-4094)"""
        return 1 <= vlan_id <= 4094
    
    def validate_bgp_as(self, asn: int) -> bool:
        """Validate BGP AS number"""
        return 1 <= asn <= 4294967294
    
    def validate_host_vars(self, filepath: Path) -> List[str]:
        """Validate host variables file"""
        errors = []
        
        try:
            with open(filepath) as f:
                data = yaml.safe_load(f)
        except yaml.YAMLError as e:
            return [f"YAML parse error in {filepath}: {e}"]
        
        if not data:
            return []
        
        # Validate VLANs
        for vlan in data.get('vlans', []):
            if 'id' in vlan:
                if not self.validate_vlan_id(vlan['id']):
                    errors.append(f"Invalid VLAN ID {vlan['id']} in {filepath}")
            
            if 'ip' in vlan:
                if not self.validate_ip_address(vlan['ip']):
                    errors.append(f"Invalid IP {vlan['ip']} for VLAN in {filepath}")
        
        # Validate BGP
        if 'bgp_as' in data:
            if not self.validate_bgp_as(data['bgp_as']):
                errors.append(f"Invalid BGP AS {data['bgp_as']} in {filepath}")
        
        # Validate BGP peers
        for peer in data.get('bgp_peers', {}).values():
            if 'address' in peer:
                if not self.validate_ip_address(peer['address']):
                    errors.append(f"Invalid BGP peer IP {peer['address']} in {filepath}")
        
        return errors
    
    def validate_all(self, config_dir: str) -> Tuple[List[str], List[str]]:
        """Validate ทุก config files"""
        all_errors = []
        all_warnings = []
        
        config_path = Path(config_dir)
        
        # Validate host_vars
        for filepath in (config_path / "host_vars").glob("*.yml"):
            errors = self.validate_host_vars(filepath)
            all_errors.extend(errors)
        
        # Check for duplicate VLAN IDs per router
        # ... additional checks
        
        return all_errors, all_warnings


def main():
    validator = MikroTikConfigValidator()
    errors, warnings = validator.validate_all(".")
    
    if warnings:
        for w in warnings:
            print(f"WARNING: {w}")
    
    if errors:
        for e in errors:
            print(f"ERROR: {e}")
        sys.exit(1)
    
    print("Validation passed!")
    sys.exit(0)

if __name__ == "__main__":
    main()
```

---

## 3. Pipeline Design {#pipeline}

```yaml
# .github/workflows/network-cicd.yml
name: Network Configuration CI/CD

on:
  push:
    branches: [main, develop]
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
          pip install ansible yamllint ansible-lint pyyaml netmiko
          ansible-galaxy collection install community.routeros
      
      - name: Validate YAML syntax
        run: yamllint -c .yamllint.yml .
      
      - name: Validate configurations
        run: python scripts/validate_config.py
      
      - name: Ansible syntax check
        run: ansible-playbook --syntax-check playbooks/deploy.yml
      
      - name: Ansible lint
        run: ansible-lint playbooks/

  test-staging:
    needs: validate
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    environment: staging
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        env:
          ANSIBLE_VAULT_PASSWORD: ${{ secrets.VAULT_PASSWORD }}
        run: |
          ansible-playbook -i inventory/staging.yml \
            playbooks/deploy.yml \
            --vault-password-file <(echo "$ANSIBLE_VAULT_PASSWORD")
      
      - name: Run connectivity tests
        run: python tests/test_connectivity.py --env staging
      
      - name: Run service tests
        run: python tests/test_services.py --env staging

  deploy-production:
    needs: test-staging
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to production
        env:
          ANSIBLE_VAULT_PASSWORD: ${{ secrets.VAULT_PASSWORD }}
        run: |
          ansible-playbook -i inventory/production.yml \
            playbooks/deploy.yml \
            --vault-password-file <(echo "$ANSIBLE_VAULT_PASSWORD") \
            --diff
      
      - name: Post-deploy validation
        run: python tests/test_connectivity.py --env production
      
      - name: Notify Slack
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          fields: repo,message,commit,author
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
        if: always()
```

---

## 4. Automated Testing {#testing}

```python
# tests/test_connectivity.py

import argparse
import sys
from netmiko import ConnectHandler
from typing import List, Dict
import yaml

class NetworkTest:
    """Automated network connectivity tests"""
    
    def __init__(self, inventory: dict):
        self.inventory = inventory
        self.results = []
    
    def test_ping(self, router: dict, target: str) -> dict:
        """Test ping connectivity"""
        with ConnectHandler(**router) as conn:
            output = conn.send_command(f"/ping {target} count=3")
            
            success = "3 packets transmitted, 3 received" in output
            return {
                "test": f"ping {target}",
                "router": router["host"],
                "status": "PASS" if success else "FAIL",
                "output": output[:200]
            }
    
    def test_bgp_established(self, router: dict, peer_name: str) -> dict:
        """Test BGP peer state"""
        with ConnectHandler(**router) as conn:
            output = conn.send_command(f"/routing bgp peer print where name={peer_name}")
            
            success = "established" in output.lower()
            return {
                "test": f"bgp-peer-{peer_name}",
                "router": router["host"],
                "status": "PASS" if success else "FAIL",
                "output": output[:200]
            }
    
    def test_interface_up(self, router: dict, interface: str) -> dict:
        """Test interface is running"""
        with ConnectHandler(**router) as conn:
            output = conn.send_command(f"/interface print where name={interface}")
            
            success = "R" in output  # R = Running flag
            return {
                "test": f"interface-{interface}",
                "router": router["host"],
                "status": "PASS" if success else "FAIL"
            }
    
    def run_all_tests(self) -> List[dict]:
        results = []
        
        for test_case in self.inventory.get("tests", []):
            router = test_case["router"]
            
            for test in test_case.get("connectivity", []):
                result = self.test_ping(router, test["target"])
                results.append(result)
            
            for peer in test_case.get("bgp_peers", []):
                result = self.test_bgp_established(router, peer)
                results.append(result)
            
            for iface in test_case.get("interfaces", []):
                result = self.test_interface_up(router, iface)
                results.append(result)
        
        return results


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--env", choices=["staging", "production"], required=True)
    args = parser.parse_args()
    
    with open(f"tests/{args.env}_tests.yml") as f:
        inventory = yaml.safe_load(f)
    
    tester = NetworkTest(inventory)
    results = tester.run_all_tests()
    
    passed = sum(1 for r in results if r["status"] == "PASS")
    failed = sum(1 for r in results if r["status"] == "FAIL")
    
    print(f"\nTest Results: {passed} passed, {failed} failed")
    
    for result in results:
        status_icon = "✓" if result["status"] == "PASS" else "✗"
        print(f"{status_icon} [{result['router']}] {result['test']}")
    
    if failed > 0:
        sys.exit(1)

if __name__ == "__main__":
    main()
```

---

## 5. Rollback Strategy {#rollback}

```python
# scripts/rollback.py

import subprocess
from datetime import datetime
import librouteros

def rollback_router(host: str, username: str, password: str, 
                     git_ref: str = "HEAD~1"):
    """Rollback configuration to previous Git commit"""
    
    # Get previous config from git
    result = subprocess.run(
        ["git", "show", f"{git_ref}:host_vars/{host}.yml"],
        capture_output=True, text=True
    )
    
    if result.returncode != 0:
        raise Exception(f"Cannot get config for {host} at {git_ref}")
    
    import yaml
    config = yaml.safe_load(result.stdout)
    
    # Connect and apply previous config
    conn = librouteros.connect(host=host, username=username, password=password)
    
    # Apply rollback
    print(f"Rolling back {host} to {git_ref}")
    
    # ... apply config changes ...
    
    conn.close()
    print(f"Rollback complete for {host}")


if __name__ == "__main__":
    import sys
    host = sys.argv[1] if len(sys.argv) > 1 else "core-01"
    rollback_router(host, "admin", "password")
```

---

## 6. Lab: Network GitOps {#lab}

### Lab Steps

```bash
# Step 1: Initialize Git repo
git init network-configs
cd network-configs
git remote add origin https://github.com/myorg/network-configs.git

# Step 2: Create inventory and vars
mkdir -p inventory host_vars group_vars playbooks tests

# Step 3: Create basic playbook
cat > playbooks/deploy.yml <<'EOF'
---
- hosts: mikrotik_all
  gather_facts: no
  tasks:
    - name: Apply base configuration
      include_role:
        name: mikrotik_base
EOF

# Step 4: Commit and push
git add .
git commit -m "Initial network configuration"
git push origin main

# Step 5: Create PR for change
git checkout -b feature/add-vlan-999
# Make changes...
git commit -m "Add VLAN 999 for testing"
git push origin feature/add-vlan-999
# Create Pull Request → CI/CD runs automatically
```

### Verification Checklist

- [ ] Git repo initialized
- [ ] CI/CD pipeline configured
- [ ] Validation scripts run on PR
- [ ] Staging deploy works
- [ ] Test suites pass
- [ ] Production deploy approved
- [ ] Rollback tested

---

## Summary

Part นี้ครอบคลุม:
- **GitOps** workflow สำหรับ network
- **Config validation** automation
- **CI/CD pipeline** ด้วย GitHub Actions
- **Automated testing** connectivity
- **Rollback strategy**

---

[← Part 87: Prometheus Grafana](part-087-prometheus-grafana.md) | [Part 89: Disaster Recovery →](part-089-disaster-recovery.md)
