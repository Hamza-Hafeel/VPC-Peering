# AWS VPC Peering: Private Connectivity & Bastion Architecture

![AWS](https://img.shields.io/badge/AWS-ap--south--2-232F3E?logo=amazon-aws&logoColor=white)
![EC2](https://img.shields.io/badge/Amazon-EC2-FF9900?logo=amazonec2&logoColor=white)
![VPC Peering](https://img.shields.io/badge/VPC-Peering-0284C7)
![Status](https://img.shields.io/badge/Lab_Status-Verified_&_Tested-16A34A)

A complete AWS networking project demonstrating private, bidirectional cross-VPC communication using **VPC Peering**, custom route tables, least-privilege security groups, and an administrative **SSH ProxyJump Bastion Host**.

---

## 📌 Executive Summary

* **Student:** Hamza Hafeel
* **AWS Region:** Asia Pacific (Hyderabad) — `ap-south-2`
* **AWS Account ID:** `453732174454`
* **Date Completed:** 20 October 2026
* **Program:** EWB Cloud Labs — AWS Architect Associate (Project 01)

### Key Objectives
1. Design and deploy two non-overlapping VPC networks (`10.10.0.0/16` and `10.20.0.0/16`).
2. Isolate backend workloads (`ewb-app-a` and `ewb-app-b`) within private subnets with **0 public IPv4 addresses**.
3. Establish and activate a direct VPC Peering connection (`ewb-a-b-peer`).
4. Configure directional route tables to satisfy both forward and return traffic paths.
5. Provide secure external administrative access via an OpenSSH `ProxyJump` bastion without copying private keys to the cloud.
6. Validate connectivity through negative baseline testing, positive bidirectional testing, and a controlled security group failure/recovery drill.

---

## 🏛 Architecture Diagram

```text
               +-------------------------------------------+
               |        Administrator Workstation          |
               |          IP: 49.37.171.106/32             |
               +---------------------+---------------------+
                                     |
                                     | SSH (TCP 22)
                                     v
                       +---------------------------+
                       |   Internet Gateway (IGW)  |
                       |         ewb-igw-a         |
                       +-------------+-------------+
                                     |
+------------------------------------|------------------------------------+
| VPC A: ewb-vpc-a (10.10.0.0/16)    |                                    |
|                                    v                                    |
|   +-------------------------------------------------+                   |
|   | Public Subnet: ewb-a-public (10.10.1.0/24)      |                   |
|   | ewb-bastion (10.10.1.10)                        |                   |
|   | Public IP: 18.60.91.116                         |                   |
|   | SG: TCP 22 from Admin Public IP /32             |                   |
|   +-----------------------+-------------------------+                   |
|                           | (SSH ProxyJump Tunnel)                      |
|                           |                                             |
|   +-----------------------v-------------------------+                   |
|   | Private Subnet: ewb-a-private (10.10.2.0/24)    |                   |
|   | ewb-app-a (10.10.2.10)                          |                   |
|   | No Public IP                                    |                   |
|   | SG: TCP 22 from 10.10.1.10/32                   |                   |
|   |     ICMP Echo from 10.20.1.10/32                |                   |
|   +-----------------------+-------------------------+                   |
+---------------------------|---------------------------------------------+
                            |
                            | VPC Peering Link: ewb-a-b-peer
                            | ID: pcx-03af8378d3fd60f65
                            |
+---------------------------|---------------------------------------------+
| VPC B: ewb-vpc-b (10.20.0.0/16)                                         |
| (No Internet Gateway Attached)                                          |
|                                                                         |
|   +-----------------------v-------------------------+                   |
|   | Private Subnet: ewb-b-private (10.20.1.0/24)    |                   |
|   | ewb-app-b (10.20.1.10)                          |                   |
|   | No Public IP                                    |                   |
|   | SG: TCP 22 from 10.10.1.10/32                   |                   |
|   |     ICMP Echo from 10.10.2.10/32                |                   |
|   +-------------------------------------------------+                   |
+-------------------------------------------------------------------------+
```

---

## 📋 Deployed Resource Inventory

| Resource Name | Type | Actual AWS Resource ID | IPv4 CIDR / Assigned IP |
| :--- | :--- | :--- | :--- |
| **`ewb-vpc-a`** | VPC (Requester) | `vpc-074a3adcf92fbc0f0` | `10.10.0.0/16` |
| **`ewb-vpc-b`** | VPC (Accepter) | `vpc-0ac886432445110a9` | `10.20.0.0/16` |
| **`ewb-a-public`** | Public Subnet | `subnet-064f02f654f4e75...` | `10.10.1.0/24` (AZ: `ap-south-2b`) |
| **`ewb-a-private`** | Private Subnet | `subnet-0d386a235c0ed4...` | `10.10.2.0/24` (AZ: `ap-south-2b`) |
| **`ewb-b-private`** | Private Subnet | `subnet-04401ea6d4e850...` | `10.20.1.0/24` (AZ: `ap-south-2b`) |
| **`ewb-igw-a`** | Internet Gateway | `igw-...` | Attached to `ewb-vpc-a` |
| **`ewb-a-b-peer`** | VPC Peering Connection | `pcx-03af8378d3fd60f65` | Status: `Active` |
| **`ewb-rt-a-public`** | Route Table | `rtb-0d500e5137b8f8b5e` | Associated to `ewb-a-public` |
| **`ewb-rt-a-private`**| Route Table | `rtb-08f37f1e4dde757e8` | Associated to `ewb-a-private` |
| **`ewb-rt-b-private`**| Route Table | `rtb-00ec64c8de0a6ed09` | Associated to `ewb-b-private` |
| **`ewb-sg-bastion`** | Security Group | `sg-...` | Inbound TCP 22 (`49.37.171.106/32`) |
| **`ewb-sg-app-a`** | Security Group | `sg-0a40c302570f61925` | Inbound SSH (`10.10.1.10/32`), ICMP (`10.20.1.10/32`) |
| **`ewb-sg-app-b`** | Security Group | `sg-07664b2164498c230` | Inbound SSH (`10.10.1.10/32`), ICMP (`10.10.2.10/32`) |
| **`ewb-bastion`** | EC2 (`t3.micro`) | `i-0d0046a7abdf387f2` | Private: `10.10.1.10` \| Public: `18.60.91.116` |
| **`ewb-app-a`** | EC2 (`t3.micro`) | `i-05c2641e007a3f3de` | Private: `10.10.2.10` \| Public: *None* |
| **`ewb-app-b`** | EC2 (`t3.micro`) | `i-0cd9bb4f269a42c92` | Private: `10.20.1.10` \| Public: *None* |

---

## 🛣 Route Table Configuration & Logic

| Route Table | Destination | Target | Purpose |
| :--- | :--- | :--- | :--- |
| **`ewb-rt-a-public`** | `10.10.0.0/16`<br>`0.0.0.0/0`<br>`10.20.0.0/16` | `local`<br>`ewb-igw-a`<br>`pcx-03af8378d3fd60f65` | Local VPC A traffic<br>Direct inbound/outbound internet access for Bastion<br>**ProxyJump Egress:** Bastion SSH to App-B |
| **`ewb-rt-a-private`** | `10.10.0.0/16`<br>`10.20.0.0/16` | `local`<br>`pcx-03af8378d3fd60f65` | Local VPC A traffic<br>**Forward Path:** Routes App-A packets to VPC B |
| **`ewb-rt-b-private`** | `10.20.0.0/16`<br>`10.10.0.0/16` | `local`<br>`pcx-03af8378d3fd60f65` | Local VPC B traffic<br>**Return Path:** Routes replies back to VPC A |

> **Why does `ewb-rt-a-public` need a peering route?**  
> OpenSSH `ProxyJump` makes the Bastion host originate an independent SSH socket connection to `10.20.1.10`. By Longest Prefix Match (LPM), traffic to `10.20.0.0/16` takes the peering connection instead of the default `0.0.0.0/0` route to the internet gateway.

---

## 🔒 Security Group Specifications

All security rules implement the **Principle of Least Privilege** using explicit `/32` single-host IP addresses:

### `ewb-sg-bastion` (VPC A)
* **Inbound:** `SSH (TCP 22)` | Source: `49.37.171.106/32` (Administrator Workstation only)
* **Outbound:** `All Traffic` | Destination: `0.0.0.0/0`

### `ewb-sg-app-a` (`sg-0a40c302570f61925` - VPC A)
* **Inbound:** `SSH (TCP 22)` | Source: `10.10.1.10/32` (Bastion private IP only)
* **Inbound:** `Custom ICMP - IPv4 Echo Request` | Source: `10.20.1.10/32` (`ewb-app-b` only)
* **Outbound:** `All Traffic` | Destination: `0.0.0.0/0`

### `ewb-sg-app-b` (`sg-07664b2164498c230` - VPC B)
* **Inbound:** `SSH (TCP 22)` | Source: `10.10.1.10/32` (Bastion private IP only)
* **Inbound:** `Custom ICMP - IPv4 Echo Request` | Source: `10.10.2.10/32` (`ewb-app-a` only)
* **Outbound:** `All Traffic` | Destination: `0.0.0.0/0`

---

## 💻 Workstation SSH ProxyJump Setup

To prevent security risks associated with storing private SSH keys on intermediary jump servers, OpenSSH `ProxyJump` was configured locally on the client machine.

### 1. Windows NTFS Key Permission Hardening
```powershell
# Reset and remove inherited permissions
icacls "C:\Users\hamza\Keys\ewb-lab.pem" /inheritance:r

# Grant read/write access solely to current user
icacls "C:\Users\hamza\Keys\ewb-lab.pem" /grant:r "$($env:USERNAME):(R,W)"
```

### 2. Client Configuration (`~/.ssh/config`)
```ssh-config
Host ewb-bastion
    HostName 18.60.91.116
    User ec2-user
    IdentityFile C:\Users\hamza\Keys\ewb-lab.pem
    IdentitiesOnly yes

Host ewb-app-a
    HostName 10.10.2.10
    User ec2-user
    IdentityFile C:\Users\hamza\Keys\ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion

Host ewb-app-b
    HostName 10.20.1.10
    User ec2-user
    IdentityFile C:\Users\hamza\Keys\ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion
```

---

## 🧪 Verification & Test Results

### Test 01: Baseline Negative Test (Before Peering & Routes)
Inside `ewb-app-a` before activating peering routes:
```bash
[ec2-user@ip-10-10-2-10 ~]$ ping -c 4 10.20.1.10
PING 10.20.1.10 (10.20.1.10) 56(84) bytes of data.

--- 10.20.1.10 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3090ms
```
*Result:* **PASS (Expected Failure)** — Confirmed 0 cross-VPC reachability prior to peering.

---

### Test 02: Forward Path Success (App-A to App-B)
Inside `ewb-app-a` after routes were configured:
```bash
[ec2-user@ip-10-10-2-10 ~]$ ping -c 4 10.20.1.10
PING 10.20.1.10 (10.20.1.10) 56(84) bytes of data.
64 bytes from 10.20.1.10: icmp_seq=1 ttl=127 time=0.254 ms
64 bytes from 10.20.1.10: icmp_seq=2 ttl=127 time=0.216 ms
64 bytes from 10.20.1.10: icmp_seq=3 ttl=127 time=0.206 ms
64 bytes from 10.20.1.10: icmp_seq=4 ttl=127 time=0.225 ms

--- 10.20.1.10 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3095ms
rtt min/avg/max/mdev = 0.206/0.225/0.254/0.017 ms
```
*Result:* **PASS** — 0% packet loss, sub-millisecond cross-VPC latency.

---

### Test 03 & 04: Private SSH Jump & Reverse Path Success (App-B to App-A)
Connecting to App-B directly from the laptop via `ProxyJump`:
```powershell
PS C:\Users\hamza> ssh ewb-app-b
[ec2-user@ip-10-20-1-10 ~]$ ping -c 4 10.10.2.10
PING 10.10.2.10 (10.10.2.10) 56(84) bytes of data.
64 bytes from 10.10.2.10: icmp_seq=1 ttl=127 time=0.391 ms
64 bytes from 10.10.2.10: icmp_seq=2 ttl=127 time=0.222 ms
64 bytes from 10.10.2.10: icmp_seq=3 ttl=127 time=0.235 ms
64 bytes from 10.10.2.10: icmp_seq=4 ttl=127 time=0.205 ms

--- 10.10.2.10 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3100ms
rtt min/avg/max/mdev = 0.205/0.263/0.391/0.074 ms
```
*Result:* **PASS** — SSH ProxyJump established seamlessly; bidirectional reverse path verified.

---

### Test 05: Controlled Failure & Recovery Drill
1. **Failure Trigger:** Removed the inbound ICMP Echo Request rule (`10.10.2.10/32`) on `ewb-sg-app-b`.
   ```bash
   [ec2-user@ip-10-10-2-10 ~]$ ping -c 4 10.20.1.10
   --- 10.20.1.10 ping statistics ---
   4 packets transmitted, 0 received, 100% packet loss, time 3087ms
   ```
2. **Recovery Trigger:** Restored the inbound ICMP rule on `ewb-sg-app-b`.
   ```bash
   [ec2-user@ip-10-10-2-10 ~]$ ping -c 4 10.20.1.10
   --- 10.20.1.10 ping statistics ---
   4 packets transmitted, 4 received, 0% packet loss, time 3151ms
   rtt min/avg/max/mdev = 0.301/0.329/0.341/0.016 ms
   ```
*Result:* **PASS** — Verified stateful security group enforcement across the peering link.

---

## 💡 Troubleshooting & Technical Insights

1. **Stateful Security Groups vs. Initial Outbound Packets:**  
   During initial testing, reverse ping (`App-B -> App-A`) succeeded while forward ping (`App-A -> App-B`) suffered 100% packet loss. While security groups are stateful (inbound replies to permitted outbound queries are allowed automatically), an instance cannot originate a connection if its **outbound** rules are blocked. Diagnosing and restoring `0.0.0.0/0` outbound traffic on `ewb-sg-app-a` immediately resolved the asymmetric failure.
2. **Return Route Requirement:**  
   Packet routing across AWS VPCs is strictly non-symmetric. Even with an `Active` peering connection and an outbound route in VPC A, packets cannot be acknowledged unless VPC B contains a reciprocal route entry pointing `10.10.0.0/16` back to `pcx-03af8378d3fd60f65`.
3. **Non-Transitive Peering Limitation:**  
   AWS VPC Peering does **not** support edge-to-edge routing or transitive routing (e.g., VPC B cannot access the internet through VPC A's Internet Gateway).

---

## 🧹 Teardown & Clean-up

To avoid unnecessary AWS billing, resources were deleted in reverse dependency order:
```bash
# 1. Terminate compute instances
aws ec2 terminate-instances --instance-ids i-0d0046a7abdf387f2 i-05c2641e007a3f3de i-0cd9bb4f269a42c92

# 2. Delete VPC Peering connection (automatically cleans up peering route entries)
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id pcx-03af8378d3fd60f65

# 3. Disassociate and delete custom route tables, subnets, and internet gateway
# 4. Delete security groups and VPCs (vpc-074a3adcf92fbc0f0, vpc-0ac886432445110a9)
```

---

## 📄 License
This project was implemented as part of the **EWB Cloud Labs AWS Architect Associate** curriculum. All rights reserved.