# Part 84: Terraform สำหรับ MikroTik

## สารบัญ
1. [Terraform Overview](#overview)
2. [MikroTik Provider](#provider)
3. [Infrastructure as Code](#iac)
4. [State Management](#state)
5. [Modules](#modules)
6. [CI/CD Pipeline](#cicd)
7. [Lab: Terraform MikroTik IaC](#lab)

---

## 1. Terraform Overview {#overview}

Terraform ช่วย define network infrastructure เป็น code และ manage state ของ configuration

### Infrastructure as Code Benefits

| Benefit | Description |
|---------|-------------|
| Version Control | Config ใน Git |
| Reproducibility | Deploy เหมือนกันทุกครั้ง |
| Collaboration | Team ทำงานร่วมกัน |
| Audit Trail | เห็นว่าใครเปลี่ยนอะไร |
| Rollback | ย้อนกลับ configuration |

---

## 2. MikroTik Provider {#provider}

```hcl
# ============================================
# Terraform MikroTik Provider Configuration
# ============================================

terraform {
  required_providers {
    routeros = {
      source  = "terraform-routeros/routeros"
      version = "~> 1.0"
    }
  }
  
  required_version = ">= 1.5.0"
}

provider "routeros" {
  hosturl  = "https://192.168.1.1"
  username = "admin"
  password = var.router_password
  insecure = true  # ถ้าใช้ self-signed cert
}

variable "router_password" {
  type        = string
  sensitive   = true
  description = "MikroTik admin password"
}
```

---

## 3. Infrastructure as Code {#iac}

```hcl
# ============================================
# Network Configuration as Code
# ============================================

# main.tf

# VLAN Interfaces
resource "routeros_interface_vlan" "vlans" {
  for_each = var.vlans

  name      = each.key
  vlan_id   = each.value.id
  interface = each.value.interface
  comment   = each.value.comment
}

# IP Addresses
resource "routeros_ip_address" "vlan_ips" {
  for_each = { for k, v in var.vlans : k => v if v.ip != null }

  address   = each.value.ip
  interface = routeros_interface_vlan.vlans[each.key].name
  comment   = "Managed by Terraform"

  depends_on = [routeros_interface_vlan.vlans]
}

# Firewall Rules
resource "routeros_ip_firewall_filter" "allow_established" {
  chain            = "input"
  connection_state = "established,related"
  action           = "accept"
  comment          = "Allow established - TF"
  place_before     = "0"
}

resource "routeros_ip_firewall_filter" "drop_invalid" {
  chain            = "input"
  connection_state = "invalid"
  action           = "drop"
  comment          = "Drop invalid - TF"
}

# DHCP Server
resource "routeros_ip_dhcp_server" "servers" {
  for_each = { for k, v in var.vlans : k => v if v.dhcp != null }

  name        = "dhcp-${each.key}"
  interface   = routeros_interface_vlan.vlans[each.key].name
  address_pool = routeros_ip_pool.pools[each.key].name
  disabled    = false

  depends_on = [routeros_interface_vlan.vlans]
}

resource "routeros_ip_pool" "pools" {
  for_each = { for k, v in var.vlans : k => v if v.dhcp != null }

  name   = "pool-${each.key}"
  ranges = each.value.dhcp.ranges
}


# BGP Configuration
resource "routeros_routing_bgp_instance" "main" {
  name      = "default"
  as        = var.bgp_as
  router_id = var.bgp_router_id
}

resource "routeros_routing_bgp_peer" "upstream" {
  for_each = var.bgp_peers

  name           = each.key
  remote_address = each.value.address
  remote_as      = each.value.as
  instance       = routeros_routing_bgp_instance.main.name
  comment        = each.value.comment
}
```

```hcl
# variables.tf

variable "vlans" {
  type = map(object({
    id        = number
    interface = string
    ip        = optional(string)
    comment   = string
    dhcp = optional(object({
      ranges  = string
      gateway = string
    }))
  }))

  default = {
    "vlan-mgmt" = {
      id        = 10
      interface = "bridge1"
      ip        = "10.0.10.1/24"
      comment   = "Management VLAN"
    }
    "vlan-users" = {
      id        = 100
      interface = "bridge1"
      ip        = "10.100.0.1/24"
      comment   = "Users VLAN"
      dhcp = {
        ranges  = "10.100.0.10-10.100.0.250"
        gateway = "10.100.0.1"
      }
    }
    "vlan-guest" = {
      id        = 200
      interface = "bridge1"
      ip        = "10.200.0.1/24"
      comment   = "Guest VLAN"
      dhcp = {
        ranges  = "10.200.0.10-10.200.0.100"
        gateway = "10.200.0.1"
      }
    }
  }
}

variable "bgp_as" {
  type    = number
  default = 65000
}

variable "bgp_router_id" {
  type    = string
  default = "10.0.0.1"
}

variable "bgp_peers" {
  type = map(object({
    address = string
    as      = number
    comment = string
  }))
  default = {}
}
```

---

## 4. State Management {#state}

```hcl
# ============================================
# Terraform State Backend
# ============================================

# Remote state บน S3 + DynamoDB (AWS)
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "mikrotik/prod/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# หรือใช้ Terraform Cloud
# terraform {
#   cloud {
#     organization = "my-org"
#     workspaces {
#       name = "mikrotik-production"
#     }
#   }
# }
```

```bash
# Terraform commands
terraform init      # Initialize
terraform plan      # Preview changes
terraform apply     # Apply changes
terraform destroy   # Destroy all managed resources

# Plan output to file
terraform plan -out=tfplan
terraform apply tfplan

# Target specific resource
terraform apply -target=routeros_interface_vlan.vlans

# Import existing resource
terraform import routeros_interface_vlan.vlans["vlan-mgmt"] vlan-mgmt

# State operations
terraform state list
terraform state show routeros_interface_vlan.vlans
terraform state rm routeros_interface_vlan.vlans["vlan-old"]
```

---

## 5. Modules {#modules}

```hcl
# modules/mikrotik_security/main.tf

variable "management_subnet" {
  type        = string
  description = "Management network CIDR"
}

variable "syslog_server" {
  type        = string
  description = "Syslog server IP"
  default     = ""
}

resource "routeros_ip_firewall_filter" "security_rules" {
  for_each = {
    "1-established" = {
      chain            = "input"
      connection_state = "established,related"
      action           = "accept"
      comment          = "Allow established"
    }
    "2-invalid" = {
      chain            = "input"
      connection_state = "invalid"
      action           = "drop"
      comment          = "Drop invalid"
    }
    "3-management" = {
      chain       = "input"
      src_address = var.management_subnet
      action      = "accept"
      comment     = "Allow management"
    }
    "4-drop-all" = {
      chain   = "input"
      action  = "drop"
      comment = "Default drop"
    }
  }

  chain            = each.value.chain
  action           = each.value.action
  comment          = "${each.value.comment} - TF"
  connection_state = try(each.value.connection_state, null)
  src_address      = try(each.value.src_address, null)
}

resource "routeros_system_logging_action" "remote" {
  count = var.syslog_server != "" ? 1 : 0

  name        = "syslog-remote"
  target      = "remote"
  remote      = var.syslog_server
  remote_port = 514
  bsd_syslog  = true
}


# modules/mikrotik_security/outputs.tf
output "rules_applied" {
  value = length(routeros_ip_firewall_filter.security_rules)
}


# Using module
# main.tf
module "security" {
  source = "./modules/mikrotik_security"

  management_subnet = "10.0.0.0/24"
  syslog_server     = "10.0.0.50"
}

output "security_rules" {
  value = module.security.rules_applied
}
```

---

## 6. CI/CD Pipeline {#cicd}

```yaml
# .github/workflows/terraform.yml
name: Terraform MikroTik

on:
  push:
    branches: [main]
    paths: ['terraform/**']
  pull_request:
    branches: [main]

env:
  TF_VERSION: "1.6.0"

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Terraform Init
        working-directory: terraform/
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: terraform init
      
      - name: Terraform Validate
        working-directory: terraform/
        run: terraform validate
      
      - name: Terraform Format Check
        working-directory: terraform/
        run: terraform fmt -check -recursive
      
      - name: Terraform Plan
        working-directory: terraform/
        env:
          TF_VAR_router_password: ${{ secrets.ROUTER_PASSWORD }}
        run: terraform plan -out=tfplan
      
      - name: Upload Plan
        uses: actions/upload-artifact@v3
        with:
          name: tfplan
          path: terraform/tfplan

  apply:
    needs: validate
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4
      
      - name: Download Plan
        uses: actions/download-artifact@v3
        with:
          name: tfplan
          path: terraform/
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
      
      - name: Terraform Apply
        working-directory: terraform/
        env:
          TF_VAR_router_password: ${{ secrets.ROUTER_PASSWORD }}
        run: terraform apply tfplan
```

---

## 7. Lab: Terraform MikroTik IaC {#lab}

### Lab Steps

```bash
# Step 1: Initialize project
mkdir mikrotik-terraform && cd mikrotik-terraform

# Step 2: Create main.tf
cat > main.tf <<'EOF'
terraform {
  required_providers {
    routeros = {
      source  = "terraform-routeros/routeros"
      version = "~> 1.0"
    }
  }
}

provider "routeros" {
  hosturl  = "https://192.168.1.1"
  username = "admin"
  password = "admin123"
  insecure = true
}

resource "routeros_interface_vlan" "test" {
  name      = "vlan-test"
  vlan_id   = 999
  interface = "ether2"
  comment   = "Test VLAN by Terraform"
}
EOF

# Step 3: Initialize and apply
terraform init
terraform plan
terraform apply

# Step 4: Verify on router
# /interface vlan print where name=vlan-test

# Step 5: Destroy
terraform destroy
```

### Verification Checklist

- [ ] Terraform init ทำงาน (provider downloaded)
- [ ] Terraform plan แสดง changes ถูกต้อง
- [ ] Terraform apply สร้าง resources
- [ ] Resources ปรากฏบน router
- [ ] State file บันทึกถูกต้อง
- [ ] Terraform destroy ลบ resources
- [ ] CI/CD pipeline ทำงาน

> **Tip:** ใช้ `terraform fmt` ก่อน commit เสมอ

> **Warning:** ระวัง `terraform destroy` จะลบ ALL managed resources

---

## Summary

Part นี้ครอบคลุม:
- **Terraform basics** และ MikroTik provider
- **Infrastructure as Code** สำหรับ VLANs, Firewall, BGP
- **State management** ด้วย remote backend
- **Modules** สำหรับ reusable config
- **CI/CD pipeline** ด้วย GitHub Actions

---

[← Part 83: Ansible](part-083-ansible.md) | [Part 85: Netmiko Python →](part-085-netmiko-python.md)
