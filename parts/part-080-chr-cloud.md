# Part 80: CHR - Cloud Hosted Router

## สารบัญ
1. [CHR Overview](#overview)
2. [Licensing](#licensing)
3. [AWS Deployment](#aws)
4. [GCP Deployment](#gcp)
5. [Azure Deployment](#azure)
6. [CHR Configuration](#configuration)
7. [Performance Tuning](#performance)
8. [Lab: CHR on AWS](#lab)

---

## 1. CHR Overview {#overview}

CHR (Cloud Hosted Router) คือ RouterOS ที่ run บน virtual machine ในทุก cloud provider

### CHR Use Cases

| Use Case | Description |
|----------|-------------|
| VPN Gateway | IPsec/OpenVPN/WireGuard termination |
| Cloud Router | Routing ระหว่าง VPCs |
| Software Defined WAN | SD-WAN hub |
| Development/Test | Lab environment |
| ISP in Cloud | Virtual ISP services |

### CHR Limitations

```
- ต้องการ x86/x86_64 platform
- ไม่รองรับ hardware acceleration (FPGA, switch chip)
- Performance ขึ้นกับ VM specs
- Layer 2 bridging มีข้อจำกัดบน cloud
```

---

## 2. Licensing {#licensing}

```
CHR License Tiers:
┌─────────────┬──────────────┬─────────────────┐
│ Tier        │ Speed Limit  │ Price           │
├─────────────┼──────────────┼─────────────────┤
│ Free/Trial  │ 1 Mbps       │ Free            │
│ P1          │ 1 Gbps       │ $45/lifetime    │
│ P10         │ 10 Gbps      │ $95/lifetime    │
│ P-Unlimited │ Unlimited    │ $250/lifetime   │
│ 60-day Free │ 1 Gbps       │ Trial           │
└─────────────┴──────────────┴─────────────────┘

License activation:
/system license renew account=mikrotik-user level=p1
```

---

## 3. AWS Deployment {#aws}

```bash
# ============================================
# Deploy CHR บน AWS EC2
# ============================================

# 1. ดาวน์โหลด CHR AMI
# https://mikrotik.com/download → Cloud Hosted Router → Raw disk image

# 2. แปลง image เป็น AMI
aws ec2 import-image \
  --description "MikroTik CHR 7.x" \
  --disk-containers "Format=RAW,UserBucket={S3Bucket=my-bucket,S3Key=chr-7.15.vmdk}"

# 3. สร้าง Security Group
aws ec2 create-security-group \
  --group-name mikrotik-sg \
  --description "MikroTik CHR Security Group"

aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxx \
  --protocol tcp --port 22 --cidr 0.0.0.0/0    # SSH

aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxx \
  --protocol tcp --port 8291 --cidr 0.0.0.0/0  # Winbox

aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxx \
  --protocol udp --port 1194 --cidr 0.0.0.0/0  # OpenVPN

aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxx \
  --protocol 50 --cidr 0.0.0.0/0               # IPsec ESP

aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxx \
  --protocol udp --port 500 --cidr 0.0.0.0/0   # IKE

aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxx \
  --protocol udp --port 4500 --cidr 0.0.0.0/0  # NAT-T

# 4. Launch instance
aws ec2 run-instances \
  --image-id ami-xxxx \
  --instance-type t3.medium \
  --key-name my-key \
  --security-group-ids sg-xxxx \
  --subnet-id subnet-xxxx \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=mikrotik-chr}]'

# 5. Assign Elastic IP
aws ec2 allocate-address --domain vpc
aws ec2 associate-address --instance-id i-xxxx --allocation-id eipalloc-xxxx
```

---

## 4. GCP Deployment {#gcp}

```bash
# ============================================
# Deploy CHR บน Google Cloud Platform
# ============================================

# 1. Upload CHR image to GCS
gsutil cp chr-7.15.img.zip gs://my-bucket/mikrotik/

# 2. Unzip and create disk image
gcloud compute images create chr-7-15 \
  --source-uri gs://my-bucket/mikrotik/chr-7.15.img \
  --description "MikroTik CHR 7.15"

# 3. Create instance
gcloud compute instances create mikrotik-chr \
  --machine-type=n1-standard-2 \
  --image=chr-7-15 \
  --zone=asia-southeast1-a \
  --tags=mikrotik-firewall \
  --can-ip-forward

# 4. Create firewall rules
gcloud compute firewall-rules create allow-mikrotik \
  --target-tags=mikrotik-firewall \
  --allow=tcp:22,tcp:8291,udp:1194,udp:500,udp:4500 \
  --source-ranges=0.0.0.0/0

# 5. Reserve static IP
gcloud compute addresses create mikrotik-ip \
  --region=asia-southeast1

gcloud compute instances add-access-config mikrotik-chr \
  --address=34.x.x.x
```

---

## 5. Azure Deployment {#azure}

```bash
# ============================================
# Deploy CHR บน Microsoft Azure
# ============================================

# 1. Upload VHD to Azure Blob Storage
az storage blob upload \
  --account-name mystorageaccount \
  --container-name vhds \
  --name chr-7.15.vhd \
  --file chr-7.15.vhd \
  --type page

# 2. Create managed disk from VHD
az disk create \
  --name chr-disk \
  --resource-group myRG \
  --location southeastasia \
  --source https://mystorageaccount.blob.core.windows.net/vhds/chr-7.15.vhd

# 3. Create image from disk
az image create \
  --name chr-image \
  --resource-group myRG \
  --os-type Linux \
  --os-disk chr-disk

# 4. Create VM from image
az vm create \
  --resource-group myRG \
  --name mikrotik-chr \
  --image chr-image \
  --size Standard_B2s \
  --location southeastasia \
  --admin-username admin \
  --no-wait

# 5. Open ports
az vm open-port --resource-group myRG --name mikrotik-chr --port 22,8291,1194
```

---

## 6. CHR Configuration {#configuration}

```bash
# ============================================
# Initial CHR Configuration
# ============================================

# First login via SSH
# Default: admin / (empty password)

# Security setup
/user set admin password=StrongPass@2024!
/ip service disable telnet,ftp,www,api

# Network interfaces
/ip address
add address=10.0.1.1/24 interface=ether1 comment="Public/WAN"
add address=172.16.0.1/24 interface=ether2 comment="Private LAN"

# NAT
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade

# Firewall
/ip firewall filter
add chain=input connection-state=established,related action=accept
add chain=input connection-state=invalid action=drop
add chain=input protocol=icmp action=accept
add chain=input dst-port=22,8291 protocol=tcp action=accept
add chain=input action=drop

# VPN Gateway (WireGuard)
/interface wireguard
add name=wg0 listen-port=51820

# หรือ OpenVPN
/interface ovpn-server server
set enabled=yes port=1194 mode=ip

# BGP (ถ้าต้องการ routing ในระบบ cloud)
/routing bgp instance
add name=default as=65000 router-id=10.0.1.1

/routing bgp peer
add name=aws-peer remote-address=169.254.0.2 remote-as=64512 \
    comment="AWS VGW"
```

---

## 7. Performance Tuning {#performance}

```bash
# ============================================
# CHR Performance Optimization
# ============================================

# เพิ่ม CPU cores
# ใช้ t3.xlarge (4 vCPUs) แทน t3.micro (2 vCPUs)

# Network driver optimization (ใช้ ENA บน AWS)
# ต้องเลือก instance type ที่รองรับ ENA

# RouterOS performance settings
/system resource cpu print
/system routerboard print

# Disable FastTrack ถ้า CPU limit
/ip firewall filter
add chain=forward action=fasttrack-connection \
    connection-state=established,related \
    comment="FastTrack established"

# Queue configuration สำหรับ throughput
/queue interface
set ether1 queue=no-queue

# ปิด features ที่ไม่ต้องการ
/ip settings set ip-forward=yes
/ipv6 settings set forward=yes disable-ipv6=no

# เพิ่ม receive queues
# (ขึ้นกับ cloud provider driver)
```

---

## 8. Lab: CHR on AWS {#lab}

### Lab Architecture

```
Internet
    │
[AWS IGW]
    │
[VPC: 10.0.0.0/16]
    │
[Public Subnet: 10.0.1.0/24]
    │
[MikroTik CHR]
  eth0: 10.0.1.10 (Public EIP: x.x.x.x)
  eth1: 10.0.2.10 (Private: 10.0.2.10)
    │
[Private Subnet: 10.0.2.0/24]
    │
[EC2 instances]
```

### Terraform Deployment

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-southeast-1"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "mikrotik-vpc" }
}

resource "aws_instance" "chr" {
  ami           = var.chr_ami_id
  instance_type = "t3.medium"
  
  network_interface {
    network_interface_id = aws_network_interface.public.id
    device_index         = 0
  }
  
  network_interface {
    network_interface_id = aws_network_interface.private.id
    device_index         = 1
  }
  
  tags = { Name = "mikrotik-chr" }
}

resource "aws_eip" "chr" {
  network_interface = aws_network_interface.public.id
}

output "chr_public_ip" {
  value = aws_eip.chr.public_ip
}
```

### Verification Checklist

- [ ] CHR instance running บน cloud
- [ ] EIP/Static IP assigned
- [ ] SSH access ทำงาน
- [ ] Winbox access ทำงาน
- [ ] NAT สำหรับ private instances
- [ ] VPN gateway configured
- [ ] Firewall rules applied
- [ ] License activated

> **Note:** AWS EC2 ต้องการ disable "Source/Destination Check" บน network interface เพื่อให้ routing ทำงาน

> **Warning:** CHR trial license จำกัด 1 Mbps - ต้อง activate license สำหรับ production

---

## Summary

Part นี้ครอบคลุม:
- **CHR overview** และ licensing
- **AWS deployment** ด้วย EC2
- **GCP deployment** บน Compute Engine
- **Azure deployment** บน Virtual Machines
- **CHR configuration** สำหรับ cloud
- **Performance tuning** บน virtual environment

---

[← Part 79: CAPsMAN](part-079-capsman.md) | [Part 81: Cloud Environments →](part-081-cloud-environments.md)
