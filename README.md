<img width="1055" height="340" alt="01-vnet-subnets" src="https://github.com/user-attachments/assets/248b564c-daf1-4298-8144-5c9ea19f0301" />
<img width="1055" height="340" alt="01-vnet-subnets" src="https://github.com/user-attachments/assets/96924733-80fe-432b-b92c-b92ec6401c1a" />
# Azure Secure Three-Tier Network

## Project Overview

This project demonstrates the design, deployment, testing, troubleshooting, and security hardening of a segmented three-tier network architecture in Microsoft Azure.

The environment separates resources into **Web, Application, and Data tiers** using Azure Virtual Networks, subnets, Network Security Groups (NSGs), and Linux virtual machines.

The primary objective was to implement **least-privilege network communication** and validate that authorized traffic was permitted while unauthorized traffic between application tiers was blocked.

---

## Architecture

```text
                         INTERNET
                             |
                         HTTPS/443
                             |
                             v
                    +----------------+
                    |    WEB TIER    |
                    | 10.10.1.0/24   |
                    |   vm-web-1     |
                    |    nsg-web     |
                    +-------+--------+
                            |
                         TCP 8080
                         ALLOWED
                            |
                            v
                    +----------------+
                    |    APP TIER    |
                    | 10.10.2.0/24   |
                    |   vm-app-1     |
                    |    nsg-app     |
                    +-------+--------+
                            |
                         TCP 1433
                         ALLOWED
                            |
                            v
                    +----------------+
                    |   DATA TIER    |
                    | 10.10.3.0/24   |
                    |  vm-data-1     |
                    |   nsg-data     |
                    +----------------+
```

## Network Design

| Tier        | Subnet         | Purpose                          |
| ----------- | -------------- | -------------------------------- |
| Web         | `10.10.1.0/24` | Web-facing application tier      |
| Application | `10.10.2.0/24` | Internal application services    |
| Data        | `10.10.3.0/24` | Protected database/data services |

All three subnets exist inside the same Azure Virtual Network.

Network Security Groups are used to control which tiers are allowed to communicate with each other.
---<img width="1055" height="340" alt="01-vnet-subnets" src="https://github.com/user-attachments/assets/eb15cc55-2cb0-491b-b455-1492502d4f5c" />


## Technologies Used

* Microsoft Azure
* Azure Virtual Network
* Azure Subnets
* Azure Network Security Groups
* Azure Linux Virtual Machines
* Azure Run Command
* TCP/IP
* CIDR subnetting
* Linux
* Python HTTP server
* `curl`
* `ss`

---

## Security Objectives

The environment was designed around the principle of **least privilege**.

Each application tier should only be able to communicate with the systems required for its function.

### Intended Traffic Flow

```text
Internet → Web      TCP 443      ALLOW
Web → App           TCP 8080     ALLOW
App → Data          TCP 1433     ALLOW

Data → App          TCP 8080     DENY
Web → Data          TCP 1433     DENY
```

The virtual machines were deployed without public IP addresses to reduce unnecessary direct Internet exposure.

---

# Network Security Groups

## Web Tier — nsg-web

The Web NSG was configured to permit HTTPS traffic to the Web tier.

```text
Protocol: TCP
Destination Port: 443
Action: Allow
Priority: 100
```

---

## Application Tier — nsg-app

The Application tier accepts TCP 8080 traffic from the Web subnet.

### Allow Rule

```text
Source:
10.10.1.0/24

Destination:
10.10.2.0/24

Protocol:
TCP

Destination Port:
8080

Action:
Allow

Priority:
100
```

A second rule prevents other resources inside the VNet from using TCP 8080 to access the Application tier.

### Deny Rule

```text
Source:
VirtualNetwork

Destination:
10.10.2.0/24

Protocol:
TCP

Destination Port:
8080

Action:
Deny

Priority:
200
```

Because priority `100` is evaluated before priority `200`, authorized Web-to-App traffic is permitted while other VNet traffic attempting to use TCP 8080 is blocked.

---

## Data Tier — nsg-data

The Data tier accepts TCP 1433 traffic from the Application subnet.

### Allow Rule

```text
Source:
10.10.2.0/24

Destination:
10.10.3.0/24

Protocol:
TCP

Destination Port:
1433

Action:
Allow

Priority:
100
```

Other VNet traffic attempting to reach the Data tier over TCP 1433 is denied.

### Deny Rule

```text
Source:
VirtualNetwork

Destination:
10.10.3.0/24

Protocol:
TCP

Destination Port:
1433

Action:
Deny

Priority:
200
```

---

# Security Validation

Configuring security rules alone does not prove that they work.

I performed connectivity testing between the Linux VMs using Azure Run Command.

Temporary Python services were created to provide real TCP listening endpoints.

---

## Application Test Service

A temporary service was started on `vm-app-1` using TCP port 8080:

```bash
nohup python3 -m http.server 8080 --bind 0.0.0.0 >/tmp/app8080.log 2>&1 &
sleep 2
ss -lnt | grep 8080
```

The VM reported a listener on:

```text
0.0.0.0:8080
```

This provided a real endpoint for Web-to-App connectivity testing.

---

## Data Test Service

A temporary service was started on `vm-data-1` using TCP port 1433:

```bash
nohup python3 -m http.server 1433 --bind 0.0.0.0 >/tmp/data1433.log 2>&1 &
sleep 2
ss -lnt | grep 1433
```

Port 1433 was selected to simulate the network path used by Microsoft SQL Server.

The Python HTTP server was used only as a temporary TCP test service and does not represent an actual SQL Server deployment.

---

# Connectivity Test Results

| Source | Destination | Port | Expected | Result    |
| ------ | ----------- | ---: | -------- | --------- |
| Web    | App         | 8080 | Allow    | ✅ PASS    |
| App    | Data        | 1433 | Allow    | ✅ PASS    |
| Data   | App         | 8080 | Deny     | ✅ BLOCKED |
| Web    | Data        | 1433 | Deny     | ✅ BLOCKED |

---

# Test 1 — Web to Application

The following command was executed from `vm-web-1`:

```bash
curl -v --connect-timeout 5 http://10.10.2.4:8080
```

The connection succeeded:

```text
Connected to 10.10.2.4 port 8080
HTTP/1.0 200 OK
```

### Result

**PASS ✅**

The Web tier successfully communicated with the Application tier over the approved TCP 8080 path.

---

# Test 2 — Application to Data

The following command was executed from `vm-app-1`:

```bash
curl -v --connect-timeout 5 http://10.10.3.4:1433
```

The Data-tier test service successfully returned an HTTP response.

### Result

**PASS ✅**

The Application tier successfully communicated with the Data tier over the approved TCP 1433 path.

---

# Test 3 — Web to Data

The following command was executed from `vm-web-1`:

```bash
curl -v --connect-timeout 5 http://10.10.3.4:1433
```

After security hardening, the connection failed:

```text
Failed to connect to 10.10.3.4 port 1433
Timeout was reached
```

### Result

**BLOCKED ✅**

The Web tier cannot bypass the Application tier and communicate directly with the Data tier over TCP 1433.

---

# Test 4 — Data to Application

The following command was executed from `vm-data-1`:

```bash
curl -v --connect-timeout 5 http://10.10.2.4:8080
```

The connection failed:

```text
Failed to connect to 10.10.2.4 port 8080
Timeout was reached
```

### Result

**BLOCKED ✅**

The Data tier cannot initiate unauthorized TCP 8080 connections to the Application tier.

---

# Security Finding

During connectivity testing, I discovered that unauthorized Web-to-Data communication was initially successful.

This occurred even though I had not created a custom rule explicitly permitting Web-to-Data communication.

## Root Cause

Azure Network Security Groups contain the following default rule:

```text
AllowVnetInBound

Priority:
65000

Source:
VirtualNetwork

Destination:
VirtualNetwork

Action:
Allow
```

My original custom rules permitted the desired traffic paths, but they did not prevent other internal VNet traffic from being accepted by Azure's default `AllowVnetInBound` rule.

As a result:

```text
Web → Data
TCP 1433
```

was initially permitted.

---

# Remediation

I added explicit deny rules at priority `200`.

Authorized application traffic is permitted first by priority `100`.

Unauthorized traffic then encounters the priority `200` deny rule before reaching Azure's default `AllowVnetInBound` rule.

```text
Priority 100
Required tier-to-tier traffic
        |
        v
      ALLOW

Priority 200
Unauthorized tier traffic
        |
        v
       DENY

Priority 65000
Azure AllowVnetInBound
```

This created enforceable least-privilege communication between the application tiers.

---

# Troubleshooting Discovery — CIDR Configuration

Another configuration issue was discovered during security validation.

The original Application-to-Data rule used:

```text
10.10.2.0
```

instead of:

```text
10.10.2.0/24
```

Without `/24`, the NSG rule represented an individual IP address rather than the entire Application subnet.

The Application VM had an address inside that subnet:

```text
10.10.2.4
```

Therefore, the VM did not match the original custom rule.

The same issue was identified in the Web-to-Application rule.

I corrected the NSG rules to use the appropriate CIDR ranges:

```text
Web subnet:
10.10.1.0/24

Application subnet:
10.10.2.0/24

Data subnet:
10.10.3.0/24
```

After correcting the CIDR ranges, authorized traffic correctly matched the priority `100` allow rules.

---

# Final Security Architecture

The completed environment enforces the following communication model:

```text
                WEB TIER
              10.10.1.0/24
                    |
                    | TCP 8080
                    | ALLOW
                    v
              APPLICATION TIER
              10.10.2.0/24
                    |
                    | TCP 1433
                    | ALLOW
                    v
                DATA TIER
              10.10.3.0/24
```

Unauthorized lateral paths are blocked:

```text
WEB ─────── X ──────> DATA
        TCP 1433
         BLOCKED

DATA ────── X ──────> APP
        TCP 8080
         BLOCKED
```

---

# Key Lessons Learned

This project reinforced several important cloud engineering and cloud security concepts:

* Azure VNets provide private network boundaries for cloud resources.
* Subnets can separate workloads into different application tiers.
* Network Security Groups control inbound and outbound network traffic.
* NSG priority determines which security rule is evaluated first.
* Lower priority numbers are evaluated before higher numbers.
* Azure's default NSG rules must be considered when designing segmentation.
* Allowing required traffic does not automatically block unwanted traffic.
* CIDR notation is critical when creating subnet-level security rules.
* Successful deployment does not automatically mean an environment is secure.
* Security controls should be validated using real network tests.
* Both positive and negative connectivity testing are important.
* Private IP addressing reduces unnecessary Internet exposure.
* Troubleshooting and validation are important parts of cloud engineering.

One of the biggest lessons from this project was:

**Design the control → Deploy the control → Test the control → Identify weaknesses → Remediate → Retest.**

---

# Project Outcome

The final environment demonstrates practical experience with:

* Azure networking
* Virtual Networks
* Subnetting
* CIDR notation
* Network Security Groups
* Linux virtual machines
* TCP/IP
* Network segmentation
* Least-privilege security
* Connectivity testing
* Security validation
* Troubleshooting
* Cloud security hardening

Instead of only configuring Azure resources, this project validates that the security controls actually enforce the intended network architecture.

---

# Future Improvements

Future versions of this project will include:

* Infrastructure as Code using Terraform
* Azure Monitor
* Log Analytics
* Azure Network Watcher
* Network traffic logging and analysis
* Microsoft Defender for Cloud
* Microsoft Sentinel
* Azure Key Vault
* Managed identities
* GitHub-based deployment automation
* Additional monitoring and alerting
