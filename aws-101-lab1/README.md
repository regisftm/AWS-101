# AWS-101 Lab 1: AWS Infrastructure Foundation

## Lab Overview

**Prerequisites:**

- AWS account with IAM user/role holding `AdministratorAccess` (or equivalent VPC + EC2 + Resource Groups permissions)
- Active region access to `ca-central-1` (Canada Central)
- **FortiFlex token** for FortiGate BYOL licensing — required for Lab 2 (obtain from your instructor before the workshop begins)

### Objective

Create the AWS networking foundation before deploying FortiGate. This "infrastructure-first" approach mirrors enterprise deployment patterns and provides a clean foundation for security appliances. In this lab, you'll build just the VPC, subnets, and Internet Gateway — route tables and their associations will be configured in Lab 2 after FortiGate is deployed.

This lab follows Fortinet's official **single FortiGate-VM** reference architecture for AWS, which uses **two subnets**: an External (public) subnet for `port1` and an Internal (private) subnet that hosts both `port2` and the protected workloads.

### What You'll Build

By the end of this lab, you will have:

- ✅ Tagging strategy in place (and an optional AWS Resource Group) in the `ca-central-1` (Canada Central) region
- ✅ Virtual Private Cloud (VPC) with `10.100.0.0/16` CIDR block
- ✅ Two subnets (External, Internal) in the same Availability Zone
- ✅ Internet Gateway attached to the VPC (required for FortiGate's public interface in Lab 2)

### Architecture

```text
Redwood-AWS-VPC (10.100.0.0/16) — ca-central-1a
├── External-Subnet   (10.100.1.0/24)  — FortiGate port1
└── Internal-Subnet   (10.100.2.0/24)  — FortiGate port2 + protected workloads

Internet Gateway: Redwood-AWS-IGW (attached to Redwood-AWS-VPC)
```

### Business Context: Redwood Industries

**Company Profile:**

- 200-employee manufacturing company
- Existing FortiGate infrastructure at headquarters
- Expanding to AWS for business applications
- Needs consistent security across hybrid environment

**Today's Scenario:**
Your network team is preparing AWS infrastructure for the first cloud workloads. The security team requires that all traffic be inspected by FortiGate, maintaining the same security posture as on-premises.

---

## Tagging Strategy (Read Before You Start)

Unlike Azure, AWS does not have a "Resource Group" container that owns resources — AWS resources live in a region/VPC and are grouped logically through **tags**. You will apply a single `Project` tag to **every resource** created in this workshop:

| Tag Key | Tag Value |
| --- | --- |
| `Project` | `Redwood-AWS-101` |

The `Name` tag is set automatically by the AWS console whenever you fill in a resource's **Name** field, so you do not need to add it manually as a tag.

Consistent tagging on `Project` powers the Resource Group's filter (Step 1), enables cost allocation reports, and makes end-of-workshop clean-up trivial — filter by `Project=Redwood-AWS-101` and delete everything in one pass. The `-101` suffix is so that subsequent workshops (AWS-102, AWS-201, …) keep their resources cleanly separated.

---

## Step 1: Create an AWS Resource Group (Optional)

AWS Resource Groups provide a tag-based view of all resources belonging to this workshop — similar in spirit to an Azure Resource Group. This step is optional but strongly recommended for easy clean-up at the end of the workshop.

> [!NOTE]
> Do not block the pop-ups in your browser for the AWS Management Console!

1. **Log in to the AWS Management Console:**
   - Navigate to <https://console.aws.amazon.com>
   - Sign in with your IAM user credentials
   - ⚠️ **Important:** In the top right corner, select **Canada (Central) ca-central-1** as your active region. All work in this workshop must be performed in `ca-central-1`.

2. **Navigate to Resource Groups:**
   - In the top search bar, type `Resource Groups`
   - Click **Resource Groups & Tag Editor**

3. **Create the Resource Group:**
   - Click **Create resource group**
   - Use the parameters below:

   | Parameter | Value |
   | --- | --- |
   | Group type | Tag based |
   | **Grouping criteria** | |
   | Resource types | All supported resource types (default) |
   | Tag Key | `Project` |
   | Tag Value | `Redwood-AWS-101` |
   | **Group details** | |
   | Group name | `Redwood-AWS-RG` |
   | Group description | `Redwood Industries AWS-101 workshop resources` |
   | **Group tags** | |
   | Key | `Name` |
   | Value | `Redwood-AWS-RG` |

4. Click **Create Group**

### Validation

- [x] Resource Group `Redwood-AWS-RG` appears under **Saved resource groups**
- [x] The group will populate automatically as you create resources with the `Project=Redwood-AWS-101` tag in the next steps

### Troubleshooting

| Issue | Solution |
| --- | --- |
| "Access denied" error | Ensure your IAM user/role has `resource-groups:CreateGroup` permission |
| Region selector disabled | Some accounts require you to opt-in to `ca-central-1` — contact admin |
| Group shows no resources | Resources appear only after they are created with the matching tag |

---

## Step 2: Create the VPC

The Virtual Private Cloud (VPC) provides the private IP address space for all AWS resources in this workshop. Think of it as your datacenter network in the cloud — it is the AWS equivalent of an Azure VNet.

1. **Navigate to the VPC console:**
   - In the top search bar, type `VPC`
   - Click **VPC** (the service result)
   - Confirm you are still in **ca-central-1** (top-right region selector)

2. **Start the VPC wizard:**
   - In the left navigation menu, click **Your VPCs**
   - Click **Create VPC**

3. **VPC settings:** use the parameters below.

   | Parameter | Value |
   | --- | --- |
   | Resources to create | VPC only |
   | Name tag | `Redwood-AWS-VPC` |
   | IPv4 CIDR block | IPv4 CIDR manual input |
   | IPv4 CIDR | `10.100.0.0/16` |
   | IPv6 CIDR block | No IPv6 CIDR block |
   | Tenancy | Default |

> [!IMPORTANT]
> Do NOT use **VPC and more** — we want to build subnets explicitly in the following steps for better control. The **Name tag** field automatically creates a `Name` tag on the VPC.

4. **Additional tags:** click **Add tag** and add the following (in addition to the `Name` tag already present):

   | Tag Key | Tag Value |
   | --- | --- |
   | `Project` | `Redwood-AWS-101` |

5. Click **Create VPC**

6. Your VPC should be created and you would be presented with its details.

### Validation

- [x] VPC `Redwood-AWS-VPC` appears in the **Your VPCs** list
- [x] State shows **Available**
- [x] IPv4 CIDR shows `10.100.0.0/16`
- [x] Region is `ca-central-1`
- [x] No subnets are associated yet

### Understanding the Address Space

**Why `10.100.0.0/16`?**

- Provides 65,536 IP addresses (`10.100.0.0` through `10.100.255.255`)
- Avoids overlap with Redwood's on-premises network (`192.168.0.0/22`)
- Follows RFC 1918 private addressing standard
- Leaves room for growth and future VPCs (could use `10.101.0.0/16`, `10.102.0.0/16`, etc.)

**CIDR notation:**

- `/16` = 65,536 IPs
- `/24` = 256 IPs (what we'll use for subnets)
- `/26` = 64 IPs (common for smaller subnets)

### Troubleshooting

| Issue | Solution |
| --- | --- |
| CIDR block is invalid | Verify format: `10.100.0.0/16` (no spaces, correct slash) |
| Overlapping CIDR error | Check for existing VPCs using the same range in this account |
| VPC stuck in "pending" | Refresh the page — AWS usually provisions the VPC in under 5 seconds |

---

## Step 3: Create the External Subnet

The External subnet will host FortiGate's `port1` Elastic Network Interface (ENI), which connects to the Internet Gateway and provides management access.

1. **Navigate to Subnets:**
   - In the left navigation menu under **Virtual private cloud**, click **Subnets**

2. **Create subnet:** click **Create subnet** and use the parameters below.

   | Parameter | Value |
   | --- | --- |
   | VPC ID | `Redwood-AWS-VPC` |
   | Subnet name | `External-Subnet` |
   | Availability Zone | `ca-central-1a` |
   | IPv4 subnet CIDR block | `10.100.1.0/24` |

> [!IMPORTANT]
> Both subnets in this lab MUST be in the same Availability Zone (`ca-central-1a`). Unlike Azure, AWS subnets are AZ-scoped — a FortiGate ENI can only attach to subnets in its own AZ. Also note that AWS reserves 5 IPs per `/24` subnet (`.0`, `.1`, `.2`, `.3`, and `.255`), leaving 251 usable.

3. **Tags:** add the standard tags.

   | Tag Key | Tag Value |
   | --- | --- |
   | `Project` | `Redwood-AWS-101` |

4. Click **Create subnet**

### Validation

- [x] `External-Subnet` appears in the subnets list
- [x] CIDR shows `10.100.1.0/24`
- [x] Availability Zone shows `ca-central-1a`
- [x] Available IPv4 addresses shows **251**

### AWS Reserved IPs Explained

AWS reserves 5 IPs in every subnet:

- **10.100.1.0:** Network address
- **10.100.1.1:** VPC router (default gateway)
- **10.100.1.2:** Reserved by AWS — the actual VPC DNS resolver lives at the VPC base CIDR + 2 (`10.100.0.2`) and is reachable from any subnet in the VPC
- **10.100.1.3:** Reserved by AWS for future use
- **10.100.1.255:** Broadcast address (not used — AWS does not support broadcast)

**Usable IPs:** `10.100.1.4` through `10.100.1.254` (251 addresses)

---

## Step 4: Create the Internal Subnet

The Internal subnet hosts FortiGate's `port2` ENI **and** the protected workloads (such as the test VM you'll deploy in Lab 3). This is the standard Fortinet single-VM pattern on AWS — FortiGate inspects all egress and ingress traffic for any instance in this subnet.

1. **Still in Subnets:** click **Create subnet** again and use the parameters below.

   | Parameter | Value |
   | --- | --- |
   | VPC ID | `Redwood-AWS-VPC` |
   | Subnet name | `Internal-Subnet` |
   | Availability Zone | `ca-central-1a` (same AZ as External) |
   | IPv4 subnet CIDR block | `10.100.2.0/24` |

2. **Tags:** add the standard tags.

   | Tag Key | Tag Value |
   | --- | --- |
   | `Project` | `Redwood-AWS-101` |

3. Click **Create subnet**

### Validation

- [x] `Internal-Subnet` appears in the subnets list
- [x] CIDR shows `10.100.2.0/24`
- [x] Availability Zone shows `ca-central-1a`
- [x] Two subnets now visible (`External-Subnet` and `Internal-Subnet`)

### Why the Internal Subnet is Critical

**Purpose:**

- FortiGate `port2` will be placed here at `10.100.2.4`
- Protected workloads (web servers, application servers, databases) will share this subnet
- A custom route table in Lab 2 will send `0.0.0.0/0` from this subnet to FortiGate `port2`, forcing all egress through the firewall for inspection

> [!IMPORTANT]
> In Lab 2, `10.100.2.4` must be assigned as a **static** private IP on the FortiGate `port2` ENI — not a DHCP lease. The Internal subnet's route table will point `0.0.0.0/0` at this exact IP, so it cannot be allowed to change.

<details>
<summary> **Why isn't there a separate "Protected" subnet (and how this differs from Azure)?**</summary>

Fortinet's official single-FortiGate-VM reference architecture for AWS uses **two subnets**: External (port1) and Internal (port2 + workloads share this subnet).

This is **different from Azure**, where Fortinet's reference design (and the matching AZ-101 workshop) uses **three subnets**: External + Internal (port2 only, transit) + Protected (workloads). On Azure, a User-Defined Route in the Protected subnet steers workload traffic to FortiGate's port2 in the Internal subnet, and Azure UDRs can override even intra-subnet routes — making subnet separation cleanly enforceable.

On AWS, the equivalent 3-subnet split would not actually improve security in this single-VM lab, because AWS subnet route tables only apply to traffic *leaving* the subnet — FortiGate's own egress (FortiGuard, updates) uses its internal routing table out `port1` and creates no forwarding loop when `port2` and workloads share `Internal-Subnet`. A dedicated "Transit" subnet on AWS only earns its keep in HA, multi-AZ, GWLB, or hub-and-spoke designs (covered in AWS-102 / AWS-201).
</details>

<details>
<summary> **East-west traffic inside `Internal-Subnet` is NOT inspected by this lab's design.**</summary>

Two EC2 instances sitting in the **same** AWS subnet communicate directly via the VPC's implicit `local` route. AWS does not allow that route to be overridden for intra-subnet traffic — no subnet route table entry can intercept traffic between two ENIs in the same subnet. So if you add a second workload to `Internal-Subnet`, traffic between it and the test VM will bypass FortiGate entirely.

This is an **AWS fabric constraint**, not a FortiGate limitation. Note the contrast with Azure, where a User-Defined Route (UDR) **can** override the intra-subnet system route and force same-subnet traffic through an NVA.

Production patterns that **do** inspect east-west on AWS — per-workload subnets with FortiGate `port2` as next-hop (using AWS "more specific routing"), AWS Gateway Load Balancer (GWLB) with GWLB endpoints, or a Transit Gateway hub-and-spoke through a centralized Inspection VPC — are covered in **AWS-102** and **AWS-201**.
</details>

> [!NOTE]
> **Why isn't there a separate "Protected" subnet (and how this differs from Azure)?**
>
> Fortinet's official single-FortiGate-VM reference architecture for AWS uses **two subnets**: External (port1) and Internal (port2 + workloads share this subnet).
>
> This is **different from Azure**, where Fortinet's reference design (and the matching AZ-101 workshop) uses **three subnets**: External + Internal (port2 only, transit) + Protected (workloads). On Azure, a User-Defined Route in the Protected subnet steers workload traffic to FortiGate's port2 in the Internal subnet, and Azure UDRs can override even intra-subnet routes — making subnet separation cleanly enforceable.
>
> On AWS, the equivalent 3-subnet split would not actually improve security in this single-VM lab, because AWS subnet route tables only apply to traffic *leaving* the subnet — FortiGate's own egress (FortiGuard, updates) uses its internal routing table out `port1` and creates no forwarding loop when `port2` and workloads share `Internal-Subnet`. A dedicated "Transit" subnet on AWS only earns its keep in HA, multi-AZ, GWLB, or hub-and-spoke designs (covered in AWS-102 / AWS-201).

> [!IMPORTANT]
> **East-west traffic inside `Internal-Subnet` is NOT inspected by this lab's design.**
>
> Two EC2 instances sitting in the **same** AWS subnet communicate directly via the VPC's implicit `local` route. AWS does not allow that route to be overridden for intra-subnet traffic — no subnet route table entry can intercept traffic between two ENIs in the same subnet. So if you add a second workload to `Internal-Subnet`, traffic between it and the test VM will bypass FortiGate entirely.
>
> This is an **AWS fabric constraint**, not a FortiGate limitation. Note the contrast with Azure, where a User-Defined Route (UDR) **can** override the intra-subnet system route and force same-subnet traffic through an NVA.
>
> Production patterns that **do** inspect east-west on AWS — per-workload subnets with FortiGate `port2` as next-hop (using AWS "more specific routing"), AWS Gateway Load Balancer (GWLB) with GWLB endpoints, or a Transit Gateway hub-and-spoke through a centralized Inspection VPC — are covered in **AWS-102** and **AWS-201**.

**Traffic Flow (this lab):**

```text
Workload (Internal-Subnet) → port2 (inspect, NAT) → port1 → Internet Gateway
Internet Gateway → port1 (inspect) → port2 → Workload (Internal-Subnet)
```

```text
Workload-A (Internal-Subnet) ─── direct ENI-to-ENI ───> Workload-B (Internal-Subnet)
                                  (NOT inspected — AWS local route)
```

---

## Step 5: Create and Attach an Internet Gateway

AWS VPCs have no implicit internet connectivity. Public IP addresses on an EC2 instance are inert until an **Internet Gateway (IGW)** is attached to the VPC and a route points to it. In Lab 2, FortiGate's `port1` will use an Elastic IP — that requires an IGW to be in place first.

> [!NOTE]
> Both major hyperscalers now require **explicit** egress configuration. Azure retired its default outbound access on **September 30, 2025** and landed a second phase **March 31, 2026** where new Azure VNets default to "private subnet" mode, so new Azure VMs must reach the internet through a NAT Gateway, a Standard SKU public IP, a Standard Load Balancer with outbound rules, or an NVA with a public IP (e.g., FortiGate `port1`). AWS has always required this explicit configuration via an Internet Gateway plus a route — the design pattern in this lab is now functionally equivalent on both clouds.

1. **Navigate to Internet Gateways:**
   - In the VPC console left navigation menu, click **Internet gateways**

2. **Create the Internet Gateway:** click **Create internet gateway** and use the parameters below.

   | Parameter | Value |
   | --- | --- |
   | Name tag | `Redwood-AWS-IGW` |

   Add the standard tags:

   | Tag Key | Tag Value |
   | --- | --- |
   | `Project` | `Redwood-AWS-101` |

   Click **Create internet gateway**.

3. **Attach the IGW to the VPC:** on the new IGW's detail page, click **Actions → Attach to VPC** and use the parameter below.

   | Parameter | Value |
   | --- | --- |
   | Available VPCs | `Redwood-AWS-VPC` |

   Click **Attach internet gateway**.

### Validation

- [x] `Redwood-AWS-IGW` shows state **Attached**
- [x] The "VPC ID" column shows `Redwood-AWS-VPC`

### Key Concept: IGW vs. Route Tables

Attaching the IGW to the VPC does **not** automatically give any subnet internet access. A subnet only becomes "public" when its associated **Route Table** contains a route such as `0.0.0.0/0 → igw-xxxxxxxx`. The next step creates that route table for `External-Subnet`. The `Internal-Subnet` route table will be built in Lab 2 because its default route must point at FortiGate's `port2` ENI (which does not exist yet).

---

## Step 6: Create and Associate the External Subnet Route Table

This route table makes `External-Subnet` a true public subnet by sending `0.0.0.0/0` to the IGW you just attached. Without it, FortiGate's `port1` (deployed in Lab 2) cannot reach the Internet for FortiFlex licence activation, FortiGuard updates, or as the egress path for inspected traffic.

> [!NOTE]
> AWS subnets default to using the VPC's **Main** route table (which only has the local route). You almost always want each subnet to have its own dedicated route table — the Main table should remain empty so any forgotten/unassociated subnet is non-routable by default rather than accidentally inheriting a permissive route.

1. **Open the Route tables console:**
   - In the VPC console left navigation menu under **Virtual private cloud**, click **Route tables**
   - Click **Create route table** and use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Name | `Redwood-AWS-RT-External` |
     | VPC | `Redwood-AWS-VPC` |
     | Tags | |
     | `Project` | `Redwood-AWS-101` |

   - Click **Create route table**

2. **Add the default route to the IGW:**
   - Open the new `Redwood-AWS-RT-External` route table
   - Click the **Routes** tab
   - Click **Edit routes → Add route** and use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Destination | `0.0.0.0/0` |
     | Target | **Internet Gateway → `Redwood-AWS-IGW`** |

   - Leave the existing `10.100.0.0/16 → local` row in place (it is implicit and immutable)
   - Click **Save changes**

3. **Associate the route table with `External-Subnet`:**
   - Click the **Subnet associations** tab
   - Click **Edit subnet associations** 
   - Select the `External-Subnet` (`10.100.1.0/24`) subnet
   - Click **Save associations**

### Validation

- [x] `Redwood-AWS-RT-External` exists in **VPC → Route tables** with two routes: `10.100.0.0/16 → local` and `0.0.0.0/0 → Redwood-AWS-IGW`
- [x] `External-Subnet` appears under **Subnet associations** for `Redwood-AWS-RT-External`
- [x] `Internal-Subnet` is **not** associated with this route table (it will get its own route table in Lab 2)
- [x] The VPC's **Main** route table now shows only the implicit `local` route and no subnet associations on `External-Subnet`

---

## Lab 1 Complete! 🎉

### What You've Accomplished

You have successfully built the AWS networking foundation for Redwood Industries:

✅ **Resource Group:** `Redwood-AWS-RG` (tag-based) in `ca-central-1`  
✅ **VPC:** `Redwood-AWS-VPC` (`10.100.0.0/16`)  
✅ **Two Subnets** (both in `ca-central-1a`):

- `External-Subnet` (`10.100.1.0/24`) — For FortiGate `port1`
- `Internal-Subnet` (`10.100.2.0/24`) — For FortiGate `port2` and protected workloads

✅ **Internet Gateway:** `Redwood-AWS-IGW` attached to `Redwood-AWS-VPC`  
✅ **External Route Table:** `Redwood-AWS-RT-External` (`0.0.0.0/0 → Redwood-AWS-IGW`) associated with `External-Subnet`

### Architecture Review

```text
              ┌──────────────────────────────┐
              │  Internet Gateway            │
              │  Redwood-AWS-IGW             │
              └──────────────┬───────────────┘
                             │  (RT-External: 0.0.0.0/0 → IGW)
                             │
┌────────────────────────────┴───────────────────────────────┐
│  Redwood-AWS-VPC  (10.100.0.0/16)  —  ca-central-1a        │
│                                                            │
│  ┌──────────────────────┐  ┌────────────────────────────┐  │
│  │ External-Subnet      │  │ Internal-Subnet            │  │
│  │ 10.100.1.0/24        │  │ 10.100.2.0/24              │  │
│  │ (FortiGate port1)    │  │ (FortiGate port2 +         │  │
│  │                      │  │  protected workloads)      │  │
│  │  RT: RT-External     │  │  RT: (built in Lab 2)      │  │
│  └──────────────────────┘  └────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

Region: `ca-central-1`  
Availability Zone: `ca-central-1a`

### Key Takeaways

1. **Infrastructure-first approach:** Building networking infrastructure before deploying security appliances mirrors enterprise workflows and makes troubleshooting easier.

2. **Two-subnet single-VM design:** This follows Fortinet's official AWS reference architecture for a single FortiGate-VM:
   - **External** = Internet-facing interface (`port1`, faces the IGW)
   - **Internal** = Inspection interface (`port2`) **and** the protected workloads share this subnet; the subnet's route table sends `0.0.0.0/0` to `port2` so all egress is inspected

3. **Address planning:** The `10.100.0.0/16` space provides:
   - 65,536 total IPs
   - Clear separation from on-prem (`192.168.x.x`)
   - Room for additional subnets as needs grow (e.g., a future HA-sync subnet)

4. **AZ awareness:** In AWS, subnets are tied to a specific Availability Zone. Placing both subnets in `ca-central-1a` is required so FortiGate's ENIs can live in the same AZ as the instance.

5. **External routing is in place; Internal routing comes later.** `External-Subnet` is now a fully functional public subnet — anything you place in it (including FortiGate's `port1` in Lab 2) can reach the Internet through the IGW. The `Internal-Subnet` route table is deliberately deferred to Lab 2 because its default route must point at FortiGate's `port2` ENI, which doesn't exist yet.

### Next Steps

You're ready for the Lab 2. Here's what's coming:

**Lab 2 — FortiGate VM Deployment & Traffic Steering**:

- Deploy a FortiGate EC2 instance from AWS Marketplace (BYOL with FortiFlex token)
- Attach two ENIs: `port1` in `External-Subnet` (with an Elastic IP), `port2` in `Internal-Subnet` with a static private IP of `10.100.2.4`
- Disable **Source/Destination check** on both ENIs
- Activate the FortiFlex licence and access the FortiGate GUI
- Create a Route Table for the `Internal-Subnet` with `0.0.0.0/0 → FortiGate port2 ENI`

### Troubleshooting Reference

If you encountered issues during Lab 1, review this troubleshooting guide:

**Issue: Subnet creation failed.**

- Check for CIDR overlap with existing subnets
- Verify VPC has sufficient address space
- Ensure the subnet CIDR is within the VPC CIDR range

**Issue: Resources in wrong region.**

- Delete and recreate in `ca-central-1`
- All resources must be in same region and AZ for this workshop

**Issue: Internet Gateway will not attach.**

- Verify the IGW is not already attached to another VPC (an IGW can only be attached to one VPC at a time)
- Detach from the other VPC first, or create a new IGW

**Still Stuck?**

- Raise your hand for instructor assistance
- Check the **CloudTrail Event history** for detailed error messages
- Verify your IAM permissions include `ec2:CreateVpc`, `ec2:CreateSubnet`, `ec2:CreateInternetGateway`, `ec2:AttachInternetGateway`

---

## Configuration Checklist

Before moving to Lab 2, verify:

- [ ] Active region is `ca-central-1` (Canada Central)
- [ ] (Optional) Resource Group `Redwood-AWS-RG` exists and is tag-based on `Project=Redwood-AWS-101`
- [ ] VPC `Redwood-AWS-VPC` has CIDR `10.100.0.0/16`
- [ ] `External-Subnet` exists with CIDR `10.100.1.0/24` in `ca-central-1a`
- [ ] `Internal-Subnet` exists with CIDR `10.100.2.0/24` in `ca-central-1a`
- [ ] Internet Gateway `Redwood-AWS-IGW` is created and **attached** to `Redwood-AWS-VPC`
- [ ] Route table `Redwood-AWS-RT-External` exists with `0.0.0.0/0 → Redwood-AWS-IGW`, associated with `External-Subnet`
- [ ] Both subnets show an **Available** state with **251** available IPv4 addresses

### AWS CLI Verification (Optional)

For advanced users comfortable with the AWS CLI (v2) and a configured profile for `ca-central-1`:

```bash
# Set defaults for the session
export AWS_DEFAULT_REGION=ca-central-1

# Verify the VPC
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=Redwood-AWS-VPC" \
  --query "Vpcs[0].{VpcId:VpcId,CIDR:CidrBlock,State:State}" \
  --output table

# Verify the subnets
aws ec2 describe-subnets \
  --filters "Name=tag:Project,Values=Redwood-AWS-101" \
  --query "Subnets[].{Name:Tags[?Key=='Name']|[0].Value,CIDR:CidrBlock,AZ:AvailabilityZone,AvailableIPs:AvailableIpAddressCount}" \
  --output table

# Verify the Internet Gateway attachment
aws ec2 describe-internet-gateways \
  --filters "Name=tag:Name,Values=Redwood-AWS-IGW" \
  --query "InternetGateways[0].{IGW:InternetGatewayId,Attachments:Attachments}" \
  --output table

# Verify the External route table, its IGW route, and its subnet association
aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=Redwood-AWS-RT-External" \
  --query "RouteTables[0].{Name:Tags[?Key=='Name']|[0].Value,Routes:Routes[].[DestinationCidrBlock,GatewayId],AssocSubnets:Associations[].SubnetId}" \
  --output json
```

Expected output should match your configuration above.

---

**End of Lab 1:**

*Next: Lab 2 — FortiGate VM Deployment & Traffic Steering*

---

## Additional Resources

**AWS Documentation:**

- Amazon VPC User Guide: <https://docs.aws.amazon.com/vpc/latest/userguide/>
- VPC Subnets: <https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html>
- Internet Gateways: <https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html>
- AWS Tagging Best Practices: <https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/tagging-best-practices.html>
- AWS Resource Groups: <https://docs.aws.amazon.com/ARG/latest/userguide/welcome.html>

**Fortinet Documentation:**

- FortiGate AWS Administration Guide: <https://docs.fortinet.com/document/fortigate-public-cloud/7.6.0/aws-administration-guide>
- Deploying FortiGate-VM on AWS: <https://docs.fortinet.com/document/fortigate-public-cloud/7.6.0/aws-administration-guide/403036/deploying-fortigate-vm-on-aws>

---

*Lab Guide Version 1.0 — April 2026*  
*Questions? Ask your instructor or refer to the main workshop materials.*
