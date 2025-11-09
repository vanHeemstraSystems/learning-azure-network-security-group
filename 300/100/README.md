# 100 - Azure Network Security Group (NSG)

This is a comprehensive “reverse engineering” breakdown of Azure NSGs, showing how they work from the portal level all the way down to the TCP/IP fundamentals!​​​​​​​​​​​​​​​​

## What’s Inside - The Complete NSG Breakdown:

### **Layer 1: What You See (Azure Portal)**

- The simple interface you click
- What Azure hides from you
- The abstraction layer

### **Layer 2: NSG Components & Structure**

Complete anatomy:

- NSG resource JSON structure
- Security rules (all properties explained)
- Default rules (the 6 you can’t delete)
- Service Tags (VirtualNetwork, Internet, Storage, etc.)
- Application Security Groups (ASGs)
- Augmented rules (multiple values)

### **Layer 3: How NSG Rules Work**

The processing logic:

- Priority-based evaluation (100-4096)
- Rule matching (all conditions must match)
- **Stateful connection tracking** (critical!)
- Subnet-level vs NIC-level attachment
- Combined evaluation (when both exist)

### **Layer 4: Under the Hood - The Technology**

The real implementation:

- Software-Defined Networking (SDN)
- Azure Fabric architecture
- Where rules are actually enforced (Hyper-V vSwitch)
- Windows Filtering Platform (WFP)
- How rules are distributed (< 1 minute typically)
- Performance optimizations (hardware offload, caching)

### **Layer 5: Mapping to TCP/IP Model**

How NSG maps to each TCP/IP layer:

```
Layer 4 (Application): ✗ NSG doesn't filter here
Layer 3 (Transport):   ✓ Port filtering, stateful tracking
Layer 2 (Internet):    ✓ IP address filtering
Layer 1 (Network):     ✗ NSG doesn't see MAC addresses
```

### **Layer 6: Packet Processing Flow**

Complete packet journey:

- Outbound: VM → NIC NSG → Subnet NSG → Internet
- Inbound: Internet → Subnet NSG → NIC NSG → VM
- With actual packet examples
- State table creation and lookup
- NSG logging flow

### **Layer 7: Physical Implementation**

What’s really happening in the datacenter:

- Physical server architecture
- Where your VM actually runs
- NSG rule storage (Azure SQL Database)
- Distribution timeline (you click “Save” → rule active in ~10 seconds)

### **Layer 8: Practical Examples**

Real-world scenarios:

1. **Basic Web Server NSG** - HTTP/HTTPS/SSH rules
1. **Three-Tier Application** - Web/App/Database with proper segmentation
1. **Using ASGs** - Before/after comparison

## Key Insights You Now Have:

**The Magic of Stateful Tracking**:

```
Outbound connection initiated:
Client → Server (NSG allows outbound)

Return traffic:
Server → Client (Auto-allowed, no inbound rule needed!)

This is stateful connection tracking at work!
```

**The Hidden Infrastructure**:

```
What you see in Portal:
"Network Security Group"

What's really running:
- Distributed across all Hyper-V hosts
- Windows Filtering Platform
- State tables on every server
- Replicated configuration
- No single point of failure
```

**TCP/IP Layer Reality**:

```
NSG operates at:
✓ Layer 2 (Internet): IP addresses
✓ Layer 3 (Transport): Ports, TCP/UDP

NSG does NOT operate at:
✗ Layer 1 (Network): MAC addresses
✗ Layer 4 (Application): URLs, HTTP headers
  (That's Azure Firewall's job)
```

## How This Helps Your Technical Assessment:

When someone shows you an Azure architecture, you can now:

**Instantly Recognize**:

- ✅ Missing NSGs (security risk!)
- ✅ Over-permissive rules (0.0.0.0/0 → bad!)
- ✅ Proper segmentation (separate NSGs per tier)
- ✅ Defense in depth (subnet + NIC NSGs)

**Intelligently Comment**:

- “This architecture lacks network segmentation between tiers”
- “Database subnet should have restrictive NSG allowing only app tier”
- “Consider using ASGs instead of IP-based rules for maintainability”
- “Default deny rule needed for explicit security posture”

## Next Azure Service

Other reverse engineered Azure networking components:

1. **Azure Virtual Network (VNet)** - The foundation everything builds on
1. **Azure Load Balancer** - How traffic distribution works
1. **Azure Application Gateway** - Layer 7 load balancing deep dive
1. **Azure VPN Gateway** - Encrypted tunnels and hybrid connectivity
1. **Azure Private Link** - How private endpoints work

Each follows the same structure: Portal → Components → Technology → TCP/IP → Physical → Examples

This gives you a complete mental model of how Azure networking actually works, not just what the documentation says!​​​​​​​​​​​​​​​​

# Reverse Engineering Azure Network Security Groups (NSG)

## From Azure Portal to TCP/IP Layers - A Complete Breakdown

A comprehensive deconstruction of how Azure NSGs work, from what you click in the portal down to the network packets on the wire.

-----

## Table of Contents

- [Layer 1: What You See (Azure Portal)](#layer-1-what-you-see-azure-portal)
- [Layer 2: NSG Components & Structure](#layer-2-nsg-components--structure)
- [Layer 3: How NSG Rules Work](#layer-3-how-nsg-rules-work)
- [Layer 4: Under the Hood - The Technology](#layer-4-under-the-hood---the-technology)
- [Layer 5: Mapping to TCP/IP Model](#layer-5-mapping-to-tcpip-model)
- [Layer 6: Packet Processing Flow](#layer-6-packet-processing-flow)
- [Layer 7: Physical Implementation](#layer-7-physical-implementation)
- [Layer 8: Practical Examples](#layer-8-practical-examples)

-----

## Layer 1: What You See (Azure Portal)

### The User Interface View

When you navigate to an NSG in the Azure Portal, here’s what you see:

```
Azure Portal → Resource Groups → Your NSG
├── Overview
├── Properties
├── Inbound security rules
├── Outbound security rules
├── Network interfaces (attached)
├── Subnets (attached)
├── Diagnostics settings
└── Locks
```

### Creating an NSG: The Simple View

**Portal Steps**:

1. Click “Create a resource”
1. Search “Network Security Group”
1. Fill in:
- Name: `MyNSG`
- Resource Group: `MyRG`
- Region: `West Europe`
1. Click “Create”

**What Just Happened?**

- Azure created a JSON ARM template
- Deployed a distributed firewall ruleset
- Made it available to attach to subnets/NICs
- Created default rules automatically

### The Abstraction

**What Azure Hides From You**:

```
Simple Portal View:
┌────────────────────────┐
│  Network Security Group │
│  - Name: MyNSG         │
│  - Region: West Europe │
│  - Rules: 6 default    │
└────────────────────────┘

What's Actually Running:
┌────────────────────────────────────────┐
│ Distributed Software-Defined Firewall   │
│ - Running on Azure fabric controllers   │
│ - Replicated across availability zones  │
│ - Evaluated on every network hop        │
│ - Stateful connection tracking          │
│ - High-performance packet filtering     │
│ - Integrated with Azure SDN             │
└────────────────────────────────────────┘
```

-----

## Layer 2: NSG Components & Structure

### Anatomy of an NSG

An NSG consists of several key components:

#### 1. NSG Resource Itself

```json
{
  "name": "MyNSG",
  "location": "westeurope",
  "type": "Microsoft.Network/networkSecurityGroups",
  "id": "/subscriptions/{sub-id}/resourceGroups/MyRG/providers/Microsoft.Network/networkSecurityGroups/MyNSG",
  "properties": {
    "securityRules": [],
    "defaultSecurityRules": [],
    "networkInterfaces": [],
    "subnets": []
  }
}
```

**Resource Properties**:

- **Name**: Unique identifier within resource group
- **Location**: Azure region (metadata only - rules enforce everywhere)
- **Resource ID**: Full Azure Resource Manager path
- **Tags**: Key-value pairs for organization

#### 2. Security Rules

Each rule is a separate object:

```json
{
  "name": "Allow-SSH",
  "properties": {
    "protocol": "Tcp",
    "sourcePortRange": "*",
    "destinationPortRange": "22",
    "sourceAddressPrefix": "10.0.0.0/16",
    "destinationAddressPrefix": "*",
    "access": "Allow",
    "priority": 100,
    "direction": "Inbound",
    "sourcePortRanges": [],
    "destinationPortRanges": [],
    "sourceAddressPrefixes": [],
    "destinationAddressPrefixes": []
  }
}
```

**Rule Properties Breakdown**:

|Property                |Purpose         |Values               |Example    |
|------------------------|----------------|---------------------|-----------|
|name                    |Rule identifier |String               |“Allow-SSH”|
|priority                |Processing order|100-4096             |100        |
|direction               |Traffic flow    |Inbound/Outbound     |Inbound    |
|access                  |Action to take  |Allow/Deny           |Allow      |
|protocol                |Network protocol|TCP/UDP/ICMP/Any     |Tcp        |
|sourceAddressPrefix     |Source IP/CIDR  |IP, CIDR, Service Tag|10.0.0.0/16|
|destinationAddressPrefix|Dest IP/CIDR    |IP, CIDR, Service Tag|*          |
|sourcePortRange         |Source port     |Port or range        |*          |
|destinationPortRange    |Dest port       |Port or range        |22         |

#### 3. Default Security Rules

Every NSG gets 6 default rules (cannot be deleted):

**Inbound Default Rules**:

```
Priority 65000: AllowVNetInBound
- Source: VirtualNetwork
- Destination: VirtualNetwork
- Protocol: Any
- Action: Allow
- Purpose: Allow traffic within VNet

Priority 65001: AllowAzureLoadBalancerInBound
- Source: AzureLoadBalancer
- Destination: *
- Protocol: Any
- Action: Allow
- Purpose: Allow health probes

Priority 65500: DenyAllInBound
- Source: *
- Destination: *
- Protocol: Any
- Action: Deny
- Purpose: Default deny (if no other rules match)
```

**Outbound Default Rules**:

```
Priority 65000: AllowVNetOutBound
- Source: VirtualNetwork
- Destination: VirtualNetwork
- Protocol: Any
- Action: Allow

Priority 65001: AllowInternetOutBound
- Source: *
- Destination: Internet
- Protocol: Any
- Action: Allow

Priority 65500: DenyAllOutBound
- Source: *
- Destination: *
- Protocol: Any
- Action: Deny
```

#### 4. Service Tags

Instead of IP addresses, you can use **Service Tags**:

```
Common Service Tags:
- VirtualNetwork: All IPs in the VNet
- Internet: Public internet
- AzureLoadBalancer: Azure load balancer IPs
- Storage: Azure Storage service IPs
- Sql: Azure SQL Database IPs
- AzureActiveDirectory: Azure AD IPs
- AzureCloud: All Azure datacenter IPs
- AzureCloud.WestEurope: West Europe Azure IPs
```

**Example Using Service Tags**:

```json
{
  "name": "Allow-Storage",
  "properties": {
    "sourceAddressPrefix": "VirtualNetwork",
    "destinationAddressPrefix": "Storage",
    "destinationPortRange": "443",
    "protocol": "Tcp",
    "access": "Allow",
    "priority": 110,
    "direction": "Outbound"
  }
}
```

**Why Service Tags Are Powerful**:

- Microsoft maintains the IP lists
- Automatically updated when Azure IPs change
- No need to manage IP ranges manually
- Regional granularity available

#### 5. Application Security Groups (ASGs)

ASGs let you group VMs logically:

```
Without ASGs (IP-based):
Rule: Allow 10.0.1.4, 10.0.1.5, 10.0.1.6 → 10.0.2.4, 10.0.2.5
(Must update rule when IPs change)

With ASGs (group-based):
Rule: Allow "WebServers" ASG → "DatabaseServers" ASG
(IPs can change, rule stays the same)
```

**ASG Structure**:

```json
{
  "name": "WebServers",
  "type": "Microsoft.Network/applicationSecurityGroups",
  "location": "westeurope",
  "properties": {}
}
```

**NSG Rule Using ASGs**:

```json
{
  "name": "Allow-Web-to-DB",
  "properties": {
    "protocol": "Tcp",
    "sourceApplicationSecurityGroups": [
      {"id": "/subscriptions/.../applicationSecurityGroups/WebServers"}
    ],
    "destinationApplicationSecurityGroups": [
      {"id": "/subscriptions/.../applicationSecurityGroups/DatabaseServers"}
    ],
    "destinationPortRange": "1433",
    "access": "Allow",
    "priority": 200,
    "direction": "Outbound"
  }
}
```

#### 6. Augmented Security Rules

Enhanced rules with multiple values:

```json
{
  "name": "Allow-Multiple-Ports",
  "properties": {
    "protocol": "Tcp",
    "sourceAddressPrefix": "10.0.1.0/24",
    "destinationAddressPrefixes": [
      "10.0.2.0/24",
      "10.0.3.0/24"
    ],
    "destinationPortRanges": [
      "80",
      "443",
      "8080-8090"
    ],
    "access": "Allow",
    "priority": 300,
    "direction": "Outbound"
  }
}
```

**Benefits**:

- One rule instead of multiple
- Easier to manage
- Cleaner ruleset
- Same performance

-----

## Layer 3: How NSG Rules Work

### Rule Processing Logic

#### Step 1: Determine Direction

```
Packet arrives → Is it Inbound or Outbound?

Inbound: Traffic coming TO the resource
Outbound: Traffic leaving FROM the resource

Perspective matters!
```

#### Step 2: Evaluate Rules by Priority

```
Priority: 100 - 4096 (lower number = higher priority)

Processing:
1. Start with priority 100
2. Check if rule matches packet
3. If match: Apply action (Allow/Deny) and STOP
4. If no match: Move to next priority
5. If no custom rules match: Apply default rules
6. Default deny (65500) catches everything else
```

**Example Processing**:

```
Packet: TCP from 10.0.1.5 to 10.0.2.10:1433

Rules evaluated in order:
Priority 100: Allow 10.0.1.0/24 → * port 22 (TCP)
  → No match (port is 1433, not 22)
  → Continue

Priority 200: Allow 10.0.1.0/24 → 10.0.2.0/24 port 1433 (TCP)
  → MATCH! 
  → Action: Allow
  → STOP processing (packet allowed)

Priorities 300, 400, etc. are never evaluated
```

### Rule Matching Logic

For a rule to match, **ALL** conditions must match:

```
Rule Conditions (ALL must match):
✓ Protocol (TCP/UDP/ICMP/Any)
✓ Source IP/CIDR
✓ Source Port
✓ Destination IP/CIDR
✓ Destination Port
✓ Direction (Inbound/Outbound)

If ANY condition doesn't match → Rule doesn't apply
```

**Match Examples**:

```
Example 1: SSH Rule
Rule: Allow TCP from 10.0.1.0/24 to * port 22

Packet: TCP from 10.0.1.5:49152 to 10.0.2.10:22
✓ Protocol: TCP (matches)
✓ Source IP: 10.0.1.5 (in 10.0.1.0/24, matches)
✓ Source Port: 49152 (rule says *, matches anything)
✓ Dest IP: 10.0.2.10 (rule says *, matches)
✓ Dest Port: 22 (matches)
→ MATCH! Packet allowed

Packet: TCP from 10.0.2.5:49152 to 10.0.2.10:22
✓ Protocol: TCP (matches)
✗ Source IP: 10.0.2.5 (NOT in 10.0.1.0/24, doesn't match)
→ NO MATCH, continue to next rule
```

### Stateful Connection Tracking

NSGs are **stateful** - they track connections:

```
Stateful Behavior:

Outbound connection initiated:
Client (10.0.1.5:49152) → Server (20.30.40.50:443)

NSG creates connection state:
{
  protocol: TCP,
  sourceIP: 10.0.1.5,
  sourcePort: 49152,
  destIP: 20.30.40.50,
  destPort: 443,
  direction: Outbound,
  state: ESTABLISHED
}

Return traffic (Server → Client):
Server (20.30.40.50:443) → Client (10.0.1.5:49152)

NSG sees this matches existing connection
→ Automatically allowed (even if no inbound rule exists!)
→ This is stateful tracking
```

**Why This Matters**:

```
Without Statefulness:
Outbound Rule: Allow 10.0.1.5 → 20.30.40.50:443
Inbound Rule: Allow 20.30.40.50:443 → 10.0.1.5:49152
(Need TWO rules)

With Statefulness:
Outbound Rule: Allow 10.0.1.5 → 20.30.40.50:443
(Only ONE rule needed - return traffic auto-allowed)
```

### Attachment Points

NSGs can attach to two places:

#### Subnet-Level NSG

```
VNet: 10.0.0.0/16
  └── Subnet: 10.0.1.0/24 (NSG attached here)
      ├── VM1: 10.0.1.4
      ├── VM2: 10.0.1.5
      └── VM3: 10.0.1.6

All traffic to/from this subnet evaluated by NSG
```

**Characteristics**:

- Applies to ALL resources in subnet
- Evaluated at subnet boundary
- Easier to manage (one NSG for many VMs)
- Coarse-grained control

#### NIC-Level NSG

```
VM: WebServer1
  └── NIC (NSG attached here)
      └── Private IP: 10.0.1.4

Only traffic to/from this specific NIC evaluated
```

**Characteristics**:

- Applies to specific VM only
- Evaluated at NIC
- Fine-grained control
- More NSGs to manage

#### Combined (Both Levels)

```
Subnet NSG + NIC NSG = BOTH evaluated!

Traffic Flow:
Inbound:  External → Subnet NSG → NIC NSG → VM
Outbound: VM → NIC NSG → Subnet NSG → External

Both must allow for traffic to pass
```

**Evaluation Order**:

```
Inbound:
1. Subnet NSG inbound rules
2. If allowed → NIC NSG inbound rules
3. If both allow → Traffic reaches VM

Outbound:
1. NIC NSG outbound rules
2. If allowed → Subnet NSG outbound rules
3. If both allow → Traffic leaves

Either can deny and traffic stops
```

-----

## Layer 4: Under the Hood - The Technology

### Software-Defined Networking (SDN)

NSGs are implemented using Azure’s SDN infrastructure:

```
Traditional Firewall:
┌──────────────┐
│ Physical Box │
│ - Hardware   │
│ - Fixed      │
│ - Single     │
└──────────────┘

Azure NSG:
┌────────────────────────────────┐
│ Distributed Software Firewall   │
│ - Runs on Azure fabric          │
│ - Scales infinitely             │
│ - Replicated everywhere         │
│ - No single point of failure    │
└────────────────────────────────┘
```

### Azure Fabric Architecture

```
Azure Region (e.g., West Europe):
├── Multiple Datacenters
│   ├── Datacenter 1
│   │   ├── Rack 1
│   │   │   ├── Physical Servers
│   │   │   │   ├── Hypervisor (Hyper-V)
│   │   │   │   │   ├── Your VM
│   │   │   │   │   └── NSG enforcement point
│   │   │   │   └── Fabric Controller Agent
│   │   ├── Rack 2
│   │   └── ...
│   └── Datacenter 2
└── Fabric Controllers (manage everything)
    └── NSG rules stored and distributed
```

### Where NSG Rules Are Enforced

**Three Enforcement Points**:

1. **Hypervisor Virtual Switch (vSwitch)**

```
Physical Server:
├── Hyper-V Hypervisor
│   ├── Virtual Switch
│   │   ├── NSG Rules Loaded Here ← Enforcement
│   │   ├── Port 1: VM1 NIC
│   │   ├── Port 2: VM2 NIC
│   │   └── Port 3: VM3 NIC
│   └── VMs running above
```

1. **Azure Host Networking**

```
Before packet enters/leaves physical network:
Packet → NSG Evaluation → Physical Network

Acts like a distributed firewall across all Azure infrastructure
```

1. **Software Load Balancer (SLB)**

```
For load-balanced traffic:
Internet → Azure SLB → NSG Check → Backend VM

SLB integrated with NSG enforcement
```

### The Technology Stack

**Components Involved**:

```
Layer 1: Azure Portal/API
- You create NSG via REST API
- ARM (Azure Resource Manager) processes

Layer 2: Control Plane
- NSG configuration stored in Azure's database
- Replicated across region

Layer 3: Fabric Controllers
- Distribute NSG rules to compute nodes
- Monitor and maintain state

Layer 4: Host Agents
- Run on every physical server
- Receive NSG updates
- Configure local enforcement

Layer 5: Virtual Switch (vSwitch)
- Windows Filtering Platform (WFP)
- Packet filtering engine
- Stateful connection tracking

Layer 6: Actual Packet Processing
- Hardware offload where possible
- Software filtering
- Connection state tables
```

### How Rules Are Distributed

```
Sequence of Events:

1. You create NSG rule in portal
   ↓
2. API call to Azure Resource Manager
   ↓
3. ARM validates and stores in database
   ↓
4. Fabric Controller notified of change
   ↓
5. Fabric Controller pushes to relevant compute nodes
   ↓
6. Host Agent receives update
   ↓
7. Host Agent programs vSwitch
   ↓
8. vSwitch applies rules to packet flow
   ↓
9. Rules active (typically < 1 minute)
```

### Windows Filtering Platform (WFP)

Azure uses Windows Filtering Platform for packet filtering:

```
WFP Architecture:
┌────────────────────────────────┐
│   Application Layer            │ ← NSG rules as filter rules
├────────────────────────────────┤
│   Filter Engine                │ ← Evaluates packets
├────────────────────────────────┤
│   Callout Drivers              │ ← Custom processing
├────────────────────────────────┤
│   Network Stack                │ ← TCP/IP stack
└────────────────────────────────┘
```

**How WFP Processes NSG Rules**:

1. **Filter Installation**:
- NSG rules converted to WFP filters
- Installed at specific layers
- Priorities maintained
1. **Packet Classification**:
- Extract 5-tuple: (source IP, source port, dest IP, dest port, protocol)
- Match against filter database
- Apply action (permit/block)
1. **State Tracking**:
- Connection table maintained
- Stateful inspection
- Return traffic auto-permitted

### Performance Optimizations

**How Azure Makes NSG Fast**:

1. **Hardware Offload**:

```
Some filtering done by NIC hardware (SR-IOV)
- Reduces CPU usage
- Line-rate performance
- Nanosecond latency
```

1. **Rule Compilation**:

```
NSG rules compiled into efficient data structures
- Hash tables for fast lookup
- Bitmap filters
- Minimal memory footprint
```

1. **Connection Caching**:

```
Once connection permitted:
- Cached in fast path
- Subsequent packets bypass full rule evaluation
- Microsecond processing
```

1. **Distributed Architecture**:

```
No centralized bottleneck:
- Each server independently enforces
- No cross-server communication needed
- Linear scaling
```

-----

## Layer 5: Mapping to TCP/IP Model

Now let’s map NSG functionality to TCP/IP layers:

### TCP/IP Layer Breakdown

```
┌─────────────────────────────────────────────────┐
│ Layer 4: Application Layer                      │
│ - HTTP, FTP, SMTP, DNS                          │
│ NSG: Can filter by application (FQDN in        │
│      Azure Firewall, not basic NSG)            │
└─────────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────────┐
│ Layer 3: Transport Layer                        │
│ - TCP, UDP                                      │
│ NSG: ✓ Filters by protocol (TCP/UDP/ICMP)     │
│      ✓ Filters by port (source/dest)           │
│      ✓ Stateful tracking (TCP connections)     │
└─────────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────────┐
│ Layer 2: Internet Layer                         │
│ - IP (IPv4/IPv6), ICMP                         │
│ NSG: ✓ Filters by IP address                   │
│      ✓ Filters by CIDR block                   │
│      ✓ ICMP protocol support                   │
└─────────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────────┐
│ Layer 1: Network Access Layer                   │
│ - Ethernet, MAC addresses                      │
│ NSG: ✗ Does NOT filter at this layer           │
│      (MAC filtering not supported)              │
└─────────────────────────────────────────────────┘
```

### Layer-by-Layer NSG Interaction

#### Layer 1: Network Access Layer (Ethernet)

**What Happens Here**:

```
Physical/Data Link Layer:
- Ethernet frames
- MAC addresses
- Physical transmission

NSG Role: NONE
- NSG doesn't see MAC addresses
- Works above this layer
- Azure handles physical networking
```

**Example Frame** (NSG doesn’t process this):

```
Ethernet Frame:
[Destination MAC][Source MAC][Type][Payload][CRC]
```

#### Layer 2: Internet Layer (IP)

**What Happens Here**:

```
IP Layer:
- IP addressing
- Routing
- Fragmentation

NSG Role: PRIMARY FILTERING LAYER
- Filters by source IP
- Filters by destination IP
- Supports CIDR blocks
- Supports IPv4 and IPv6
```

**IP Packet Structure**:

```
IP Header:
┌────────────────────────────────────┐
│ Version | IHL | Type of Service    │
│ Total Length                       │
│ Identification | Flags | Fragment  │
│ TTL | Protocol | Header Checksum   │
│ Source IP Address    ← NSG checks  │
│ Destination IP Address ← NSG checks│
│ Options (if any)                   │
└────────────────────────────────────┘
│ Data (TCP/UDP/ICMP)                │
```

**NSG Evaluation at IP Layer**:

```python
def check_ip_layer(packet, nsg_rule):
    # Extract IP header fields
    source_ip = packet.ip.src
    dest_ip = packet.ip.dst
    protocol = packet.ip.protocol  # 6=TCP, 17=UDP, 1=ICMP
    
    # Check against NSG rule
    if not ip_matches(source_ip, nsg_rule.sourceAddressPrefix):
        return NO_MATCH
    
    if not ip_matches(dest_ip, nsg_rule.destinationAddressPrefix):
        return NO_MATCH
    
    if nsg_rule.protocol != "Any" and protocol != nsg_rule.protocol:
        return NO_MATCH
    
    # IP layer matches, continue to transport layer
    return CONTINUE
```

#### Layer 3: Transport Layer (TCP/UDP)

**What Happens Here**:

```
Transport Layer:
- TCP or UDP
- Port numbers
- Connection management

NSG Role: CRITICAL FILTERING LAYER
- Filters by source port
- Filters by destination port
- Stateful TCP tracking
- UDP filtering
```

**TCP Segment Structure**:

```
TCP Header:
┌────────────────────────────────────┐
│ Source Port (16 bits) ← NSG checks │
│ Destination Port (16 bits) ← NSG c │
│ Sequence Number                    │
│ Acknowledgment Number              │
│ Flags | Window Size                │
│ Checksum | Urgent Pointer          │
│ Options (if any)                   │
└────────────────────────────────────┘
│ Data (Application payload)         │
```

**NSG Evaluation at Transport Layer**:

```python
def check_transport_layer(packet, nsg_rule):
    # Extract transport header fields
    if packet.ip.protocol == TCP:
        source_port = packet.tcp.sport
        dest_port = packet.tcp.dport
        flags = packet.tcp.flags
    elif packet.ip.protocol == UDP:
        source_port = packet.udp.sport
        dest_port = packet.udp.dport
    
    # Check against NSG rule
    if not port_matches(source_port, nsg_rule.sourcePortRange):
        return NO_MATCH
    
    if not port_matches(dest_port, nsg_rule.destinationPortRange):
        return NO_MATCH
    
    # Transport layer matches
    return MATCH
```

**Stateful TCP Tracking**:

```
TCP Three-Way Handshake:

Client → Server: SYN
NSG: Outbound rule evaluated
     Create connection entry in state table
     {src_ip, src_port, dst_ip, dst_port, proto, state=SYN_SENT}

Server → Client: SYN-ACK
NSG: Checks state table
     Finds matching connection
     Updates state: SYN_RECEIVED
     Allows (even without inbound rule)

Client → Server: ACK
NSG: Checks state table
     Updates state: ESTABLISHED
     Connection fully established

Subsequent packets:
NSG: Fast path lookup in connection table
     If connection exists: Allow
     No need to evaluate all rules again
```

#### Layer 4: Application Layer

**What Happens Here**:

```
Application Layer:
- HTTP, FTP, SMTP, DNS, etc.
- Application protocols
- Payload content

NSG Role: LIMITED
- Basic NSG: NO application-layer filtering
- Azure Firewall: YES (FQDN filtering)
- NSG: Only sees port numbers, not content
```

**HTTP Request** (NSG can’t see inside):

```
TCP Payload (Application Data):
┌────────────────────────────────────┐
│ GET /index.html HTTP/1.1           │
│ Host: www.example.com              │
│ User-Agent: Mozilla/5.0            │
│ ...                                │
└────────────────────────────────────┘

NSG sees:
- Destination port 80 (HTTP)
- TCP protocol
- Source/Dest IPs

NSG does NOT see:
- URL path (/index.html)
- Host header (www.example.com)
- HTTP method (GET)

Azure Firewall CAN see these (Layer 7 filtering)
```

### Complete Packet Processing Example

**Scenario**: Web server responds to HTTP request

```
Step 1: Application Creates Data
┌──────────────────────────┐
│ HTTP/1.1 200 OK          │ ← Layer 4 (Application)
│ Content-Type: text/html  │
│ <html>...</html>         │
└──────────────────────────┘

Step 2: TCP Adds Header
┌──────────────────────────┐
│ Source Port: 80          │ ← Layer 3 (Transport)
│ Dest Port: 49152         │   NSG checks these
│ Seq, Ack, Flags          │
├──────────────────────────┤
│ HTTP Response            │
└──────────────────────────┘

Step 3: IP Adds Header
┌──────────────────────────┐
│ Source IP: 10.0.1.5      │ ← Layer 2 (Internet)
│ Dest IP: 203.0.113.45    │   NSG checks these
│ Protocol: TCP (6)        │
├──────────────────────────┤
│ TCP Segment              │
└──────────────────────────┘

Step 4: Ethernet Adds Frame
┌──────────────────────────┐
│ Source MAC: xx:xx:...    │ ← Layer 1 (Network Access)
│ Dest MAC: yy:yy:...      │   NSG ignores this
├──────────────────────────┤
│ IP Packet                │
└──────────────────────────┘

Step 5: NSG Evaluation
Extract from packet:
- Source IP: 10.0.1.5      ← From IP header
- Dest IP: 203.0.113.45    ← From IP header
- Protocol: TCP            ← From IP header
- Source Port: 80          ← From TCP header
- Dest Port: 49152         ← From TCP header
- Direction: Outbound      ← From context

Check connection table:
- Connection exists (established earlier)
- State: ESTABLISHED
- Decision: ALLOW (fast path)

If no connection existed:
- Evaluate outbound NSG rules by priority
- Match rule: Allow 10.0.1.5 → * port 80 (TCP)
- Decision: ALLOW
- Create connection entry
```

-----

## Layer 6: Packet Processing Flow

### Complete Packet Journey with NSG

Let’s trace a complete HTTP request through the Azure networking stack with NSG enforcement:

#### Scenario Setup

```
User's Computer (203.0.113.45)
     ↓
Internet
     ↓
Azure Load Balancer (Public IP: 20.50.60.70)
     ↓
VNet: 10.0.0.0/16
  └── Subnet: 10.0.1.0/24 (Subnet NSG: SubnetNSG)
      └── VM: WebServer1
          ├── NIC (NIC NSG: NicNSG)
          └── Private IP: 10.0.1.5
```

#### Outbound Request Flow (VM → Internet)

**Step 1: Application Layer**

```
Process: nginx (web server)
Action: Generates HTTP response
Data: HTTP/1.1 200 OK + HTML content
```

**Step 2: Transport Layer (TCP)**

```
Source Port: 80 (HTTP server)
Dest Port: 49152 (client's ephemeral port)
TCP adds header with ports, seq, ack, checksum
Segment created
```

**Step 3: Network Layer (IP)**

```
Source IP: 10.0.1.5 (VM's IP)
Dest IP: 203.0.113.45 (user's IP)
IP adds header with addresses, TTL, protocol
Packet created
```

**Step 4: First NSG Check (NIC NSG)**

```
Location: VM's Network Interface
NSG: NicNSG

Extraction:
- Direction: Outbound (leaving VM)
- Protocol: TCP
- Source: 10.0.1.5:80
- Dest: 203.0.113.45:49152

State Table Check:
Connection key: (10.0.1.5:80, 203.0.113.45:49152, TCP)
Status: ESTABLISHED (from inbound request)
Decision: ALLOW (stateful tracking, fast path)

If new connection:
  Evaluate outbound rules by priority:
  Priority 100: Allow 10.0.1.0/24 → Internet port 80
  Match: YES
  Action: ALLOW
  Create state entry

Result: PASS to next stage
```

**Step 5: Second NSG Check (Subnet NSG)**

```
Location: Subnet boundary
NSG: SubnetNSG

Same evaluation:
- Check state table
- Or evaluate outbound rules
- Decision: ALLOW

Result: PASS to Azure routing
```

**Step 6: Azure SDN Routing**

```
Routing decision:
- Destination 203.0.113.45 → Internet
- Route via Azure Internet Gateway
- Source NAT applied (10.0.1.5 → 20.50.60.70)
```

**Step 7: Physical Network**

```
Packet enters Microsoft's global network
- Routed to internet edge
- Leaves Azure datacenter
- Traverses internet to user
```

#### Inbound Request Flow (Internet → VM)

**Step 1: Packet Arrives at Azure Edge**

```
Source: 203.0.113.45:49152 (user)
Dest: 20.50.60.70:80 (Azure public IP)
Protocol: TCP
Flags: SYN (new connection)
```

**Step 2: Load Balancer Processing**

```
Azure Load Balancer:
- Receives packet on public IP 20.50.60.70:80
- Health probe checked (is backend healthy?)
- Selects backend: 10.0.1.5
- DNAT: 20.50.60.70:80 → 10.0.1.5:80
- Forwards to VNet
```

**Step 3: First NSG Check (Subnet NSG)**

```
Location: Subnet boundary
NSG: SubnetNSG
Direction: Inbound (entering subnet)

Extraction:
- Source: 203.0.113.45:49152
- Dest: 10.0.1.5:80
- Protocol: TCP
- Flags: SYN

Evaluation:
Priority 100: Allow Internet → 10.0.1.0/24 port 80 (TCP)
  Source check: Internet tag matches 203.0.113.45 ✓
  Dest check: 10.0.1.5 in 10.0.1.0/24 ✓
  Port check: 80 matches ✓
  Protocol: TCP matches ✓
  Match: YES
  Action: ALLOW

Create connection entry:
{
  src: 203.0.113.45:49152,
  dst: 10.0.1.5:80,
  proto: TCP,
  state: SYN_RECEIVED,
  direction: Inbound
}

Result: PASS to NIC NSG
```

**Step 4: Second NSG Check (NIC NSG)**

```
Location: VM's NIC
NSG: NicNSG
Direction: Inbound

Evaluation:
Priority 100: Allow Internet → * port 80 (TCP)
  Match: YES
  Action: ALLOW

Update connection entry (now in both NSG state tables)

Result: PASS to VM
```

**Step 5: Hypervisor vSwitch**

```
Hyper-V virtual switch:
- Final forwarding decision
- Packet delivered to VM's virtual NIC
- Interrupt generated
- VM's network driver processes packet
```

**Step 6: VM Operating System**

```
Linux kernel network stack:
- Packet received by network driver
- IP layer processes IP header
- TCP layer processes TCP header
- SYN flag detected → TCP handshake starts
- Creates socket
- Delivers to listening process (nginx on port 80)
```

**Step 7: Application Layer**

```
nginx process:
- Accepts TCP connection
- Reads HTTP request
- Generates response
- Sends response (triggers outbound flow above)
```

### NSG Logging Flow

When NSG logging is enabled:

```
Packet processing:
1. NSG rule evaluation
     ↓
2. Decision made (Allow/Deny)
     ↓
3. If logged: Create log entry
     {
       "time": "2025-11-08T10:30:45Z",
       "systemId": "...",
       "category": "NetworkSecurityGroupEvent",
       "resourceId": "/subscriptions/.../networkSecurityGroups/MyNSG",
       "operationName": "NetworkSecurityGroupCounters",
       "properties": {
         "vnetResourceGuid": "...",
         "subnetPrefix": "10.0.1.0/24",
         "macAddress": "00-0D-3A-...",
         "ruleName": "Allow-HTTP",
         "direction": "Inbound",
         "priority": 100,
         "type": "allow",
         "conditions": {
           "protocols": "TCP",
           "sourcePortRange": "49152",
           "destinationPortRange": "80",
           "sourceIP": "203.0.113.45",
           "destinationIP": "10.0.1.5"
         }
       }
     }
     ↓
4. Log sent to:
   - Storage Account (long-term storage)
   - Log Analytics (query and analysis)
   - Event Hub (streaming to external systems)
```

-----

## Layer 7: Physical Implementation

### What’s Really Happening in the Datacenter

#### Physical Layer

```
Azure Datacenter Building:
├── Power Systems
├── Cooling Systems
└── Server Racks
    ├── Top-of-Rack (ToR) Switches
    ├── Physical Servers
    │   ├── Server 1
    │   │   ├── CPUs (Intel/AMD)
    │   │   ├── RAM (hundreds of GB)
    │   │   ├── NICs (25/100 Gbps)
    │   │   │   └── SR-IOV for VM networking
    │   │   └── Storage Controllers
    │   ├── Server 2
    │   └── Server N
    └── Management Network
```

#### Your VM on Physical Hardware

```
Physical Server:
┌─────────────────────────────────────────┐
│ Hyper-V Hypervisor (Windows Server)     │
│                                         │
│ ┌─────────────────────────────────────┐ │
│ │ Virtual Switch (vSwitch)            │ │
│ │ - NSG rules loaded here             │ │
│ │ - WFP (Windows Filtering Platform)  │ │
│ │ - Connection state tables           │ │
│ │                                     │ │
│ │ Ports:                              │ │
│ │ ├── VM1 (your VM)                   │ │
│ │ ├── VM2 (another customer)          │ │
│ │ ├── VM3 (another customer)          │ │
│ │ └── Physical NIC                    │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ ┌─────────────────┐ ┌─────────────────┐ │
│ │ VM1: WebServer1 │ │ VM2: Other VM   │ │
│ │ IP: 10.0.1.5    │ │ IP: 10.0.2.8    │ │
│ │ NSG: NicNSG     │ │ NSG: OtherNSG   │ │
│ └─────────────────┘ └─────────────────┘ │
└─────────────────────────────────────────┘
         ↓
    Physical NIC
         ↓
    ToR Switch
         ↓
    Azure Network Fabric
```

#### NSG Rule Storage

```
Azure Storage Architecture:

┌────────────────────────────────────┐
│ Azure Resource Manager (ARM)       │
│ - REST API endpoint                │
│ - Authenticates requests           │
└────────────────────────────────────┘
          ↓
┌────────────────────────────────────┐
│ Resource Provider (RP)             │
│ Microsoft.Network                  │
│ - Validates NSG operations         │
└────────────────────────────────────┘
          ↓
┌────────────────────────────────────┐
│ Azure SQL Database                 │
│ - Stores NSG configuration         │
│ - Replicated for durability        │
│ - Geo-redundant                    │
└────────────────────────────────────┘
          ↓
┌────────────────────────────────────┐
│ Fabric Controllers                 │
│ - Cluster of management VMs        │
│ - Orchestrate compute resources    │
│ - Push NSG updates                 │
└────────────────────────────────────┘
          ↓
┌────────────────────────────────────┐
│ Host Agents (on every server)      │
│ - Receive NSG updates              │
│ - Configure local enforcement      │
└────────────────────────────────────┘
```

#### Rule Distribution Timeline

```
T+0s: You click "Save" on NSG rule in portal
  ↓
T+0.1s: HTTPS POST to Azure ARM API
  ↓
T+0.2s: ARM validates JSON
  ↓
T+0.3s: Microsoft.Network RP processes
  ↓
T+0.5s: Database transaction commits
  ↓
T+1s: Fabric Controller notified
  ↓
T+2s: Fabric Controller identifies affected servers
        (Servers running VMs with this NSG attached)
  ↓
T+3s: Push message sent to Host Agents
  ↓
T+5s: Host Agent receives update
  ↓
T+6s: Host Agent validates and applies
  ↓
T+7s: vSwitch reprogrammed
  ↓
T+10s: Rule active on all servers
  ↓
T+15s: Portal shows "Succeeded"

Total time: ~10-60 seconds typically
```

-----

## Layer 8: Practical Examples

### Example 1: Basic Web Server NSG

**Scenario**: Secure a web server

```
Requirements:
- Allow HTTP (80) and HTTPS (443) from internet
- Allow SSH (22) from management subnet only
- Allow outbound to internet
- Deny all other inbound traffic
```

**NSG Configuration**:

```json
{
  "name": "WebServerNSG",
  "location": "westeurope",
  "properties": {
    "securityRules": [
      {
        "name": "Allow-HTTP",
        "properties": {
          "priority": 100,
          "direction": "Inbound",
          "access": "Allow",
          "protocol": "Tcp",
          "sourceAddressPrefix": "Internet",
          "sourcePortRange": "*",
          "destinationAddressPrefix": "*",
          "destinationPortRange": "80"
        }
      },
      {
        "name": "Allow-HTTPS",
        "properties": {
          "priority": 110,
          "direction": "Inbound",
          "access": "Allow",
          "protocol": "Tcp",
          "sourceAddressPrefix": "Internet",
          "sourcePortRange": "*",
          "destinationAddressPrefix": "*",
          "destinationPortRange": "443"
        }
      },
      {
        "name": "Allow-SSH-From-Mgmt",
        "properties": {
          "priority": 120,
          "direction": "Inbound",
          "access": "Allow",
          "protocol": "Tcp",
          "sourceAddressPrefix": "10.0.255.0/24",
          "sourcePortRange": "*",
          "destinationAddressPrefix": "*",
          "destinationPortRange": "22"
        }
      },
      {
        "name": "Deny-All-Inbound",
        "properties": {
          "priority": 4096,
          "direction": "Inbound",
          "access": "Deny",
          "protocol": "*",
          "sourceAddressPrefix": "*",
          "sourcePortRange": "*",
          "destinationAddressPrefix": "*",
          "destinationPortRange": "*"
        }
      }
    ]
  }
}
```

**How It Works**:

```
Packet 1: HTTP request from internet
Source: 203.0.113.45:51234
Dest: 10.0.1.5:80
Protocol: TCP

Evaluation (Inbound):
  Priority 100: Allow Internet → * port 80 TCP
    ✓ Source: Internet tag matches
    ✓ Dest port: 80 matches
    ✓ Protocol: TCP matches
    → MATCH! Allow packet

Packet 2: SSH from admin laptop
Source: 10.0.255.10:52000
Dest: 10.0.1.5:22
Protocol: TCP

Evaluation (Inbound):
  Priority 100: Allow Internet → * port 80 TCP
    ✗ Dest port: 22 ≠ 80
    → No match, continue
  
  Priority 110: Allow Internet → * port 443 TCP
    ✗ Dest port: 22 ≠ 443
    → No match, continue
  
  Priority 120: Allow 10.0.255.0/24 → * port 22 TCP
    ✓ Source: 10.0.255.10 in 10.0.255.0/24
    ✓ Dest port: 22 matches
    ✓ Protocol: TCP matches
    → MATCH! Allow packet

Packet 3: Malicious scan
Source: 198.51.100.20:45678
Dest: 10.0.1.5:3389 (RDP)
Protocol: TCP

Evaluation (Inbound):
  Priority 100: No match (port ≠ 80)
  Priority 110: No match (port ≠ 443)
  Priority 120: No match (source not in 10.0.255.0/24)
  Priority 4096: Deny all
    ✓ Matches everything
    → DENY packet (blocked!)
```

### Example 2: Three-Tier Application

**Scenario**: Web tier, app tier, database tier

```
Architecture:
Internet
    ↓
[Load Balancer]
    ↓
Web Tier (10.0.1.0/24) - WebTierNSG
    ↓
App Tier (10.0.2.0/24) - AppTierNSG
    ↓
DB Tier (10.0.3.0/24) - DBTierNSG
```

**Web Tier NSG** (10.0.1.0/24):

```json
{
  "name": "WebTierNSG",
  "securityRules": [
    {
      "name": "Allow-HTTP-HTTPS-Inbound",
      "priority": 100,
      "direction": "Inbound",
      "access": "Allow",
      "protocol": "Tcp",
      "sourceAddressPrefix": "Internet",
      "destinationAddressPrefix": "10.0.1.0/24",
      "destinationPortRanges": ["80", "443"]
    },
    {
      "name": "Allow-App-Tier-Outbound",
      "priority": 100,
      "direction": "Outbound",
      "access": "Allow",
      "protocol": "Tcp",
      "sourceAddressPrefix": "10.0.1.0/24",
      "destinationAddressPrefix": "10.0.2.0/24",
      "destinationPortRange": "8080"
    }
  ]
}
```

**App Tier NSG** (10.0.2.0/24):

```json
{
  "name": "AppTierNSG",
  "securityRules": [
    {
      "name": "Allow-From-Web-Tier",
      "priority": 100,
      "direction": "Inbound",
      "access": "Allow",
      "protocol": "Tcp",
      "sourceAddressPrefix": "10.0.1.0/24",
      "destinationAddressPrefix": "10.0.2.0/24",
      "destinationPortRange": "8080"
    },
    {
      "name": "Allow-To-Database",
      "priority": 100,
      "direction": "Outbound",
      "access": "Allow",
      "protocol": "Tcp",
      "sourceAddressPrefix": "10.0.2.0/24",
      "destinationAddressPrefix": "10.0.3.0/24",
      "destinationPortRange": "1433"
    }
  ]
}
```

**Database Tier NSG** (10.0.3.0/24):

```json
{
  "name": "DBTierNSG",
  "securityRules": [
    {
      "name": "Allow-From-App-Tier",
      "priority": 100,
      "direction": "Inbound",
      "access": "Allow",
      "protocol": "Tcp",
      "sourceAddressPrefix": "10.0.2.0/24",
      "destinationAddressPrefix": "10.0.3.0/24",
      "destinationPortRange": "1433"
    },
    {
      "name": "Deny-All-Outbound",
      "priority": 4096,
      "direction": "Outbound",
      "access": "Deny",
      "protocol": "*",
      "sourceAddressPrefix": "*",
      "destinationAddressPrefix": "*",
      "destinationPortRange": "*"
    }
  ]
}
```

**Traffic Flow Analysis**:

```
User Request:
User → Web Server

1. Packet: 203.0.113.45:50000 → 10.0.1.5:443
   NSG: WebTierNSG (Inbound)
   Rule: Allow-HTTP-HTTPS-Inbound (Priority 100)
   Decision: ALLOW ✓

2. Web Server → App Server
   Packet: 10.0.1.5:51000 → 10.0.2.10:8080
   
   NSG: WebTierNSG (Outbound)
   Rule: Allow-App-Tier-Outbound (Priority 100)
   Decision: ALLOW ✓
   
   NSG: AppTierNSG (Inbound)
   Rule: Allow-From-Web-Tier (Priority 100)
   Decision: ALLOW ✓

3. App Server → Database
   Packet: 10.0.2.10:52000 → 10.0.3.15:1433
   
   NSG: AppTierNSG (Outbound)
   Rule: Allow-To-Database (Priority 100)
   Decision: ALLOW ✓
   
   NSG: DBTierNSG (Inbound)
   Rule: Allow-From-App-Tier (Priority 100)
   Decision: ALLOW ✓

4. Database Response → App Server
   Packet: 10.0.3.15:1433 → 10.0.2.10:52000
   
   NSG: DBTierNSG (Outbound)
   Rule: Deny-All-Outbound (Priority 4096)
   BUT: Connection exists in state table!
   Decision: ALLOW ✓ (stateful tracking)

5. App Response → Web Server
   (Allowed by stateful tracking)

6. Web Response → User
   (Allowed by stateful tracking)
```

**Security Benefits**:

- ✅ Defense in depth (multiple NSG layers)
- ✅ Least privilege (only necessary ports)
- ✅ Segmentation (tiers isolated)
- ✅ Database protected (no outbound except responses)

### Example 3: Using ASGs for Clarity

**Before ASGs** (IP-based, hard to maintain):

```json
{
  "name": "Allow-Web-to-DB",
  "sourceAddressPrefixes": [
    "10.0.1.4",
    "10.0.1.5",
    "10.0.1.6"
  ],
  "destinationAddressPrefixes": [
    "10.0.3.10",
    "10.0.3.11"
  ],
  "destinationPortRange": "1433"
}
```

Problem: Add a server? Update the rule!

**After ASGs** (group-based, self-maintaining):

```json
{
  "name": "Allow-Web-to-DB",
  "sourceApplicationSecurityGroups": [
    {"id": ".../applicationSecurityGroups/WebServers"}
  ],
  "destinationApplicationSecurityGroups": [
    {"id": ".../applicationSecurityGroups/DatabaseServers"}
  ],
  "destinationPortRange": "1433"
}
```

Benefit: Add server to ASG, rule auto-applies!

-----

## Summary: The Complete Picture

### What You Now Understand About NSGs

**Top Layer (Portal)**:

- Click, create, configure
- Simple interface
- Hides complexity

**Middle Layer (Components)**:

- Rules with priorities
- Inbound/outbound separation
- Service tags and ASGs
- Stateful connection tracking

**Bottom Layer (Technology)**:

- Windows Filtering Platform
- Hyper-V vSwitch enforcement
- Azure SDN fabric
- Distributed architecture

**TCP/IP Mapping**:

- Layer 2 (IP): IP address filtering
- Layer 3 (Transport): Port filtering, stateful tracking
- Layer 4 (Application): No filtering (Azure Firewall does this)

### Key Insights

**NSG is NOT**:

- ❌ A physical firewall appliance
- ❌ Centralized (it’s distributed)
- ❌ Layer 7 aware (can’t see URLs, etc.)
- ❌ Rate limiting
- ❌ DDoS protection

**NSG IS**:

- ✅ Software-defined distributed firewall
- ✅ Stateful packet filter
- ✅ Layer 3/4 enforcement
- ✅ Micro-segmentation enabler
- ✅ Free (no additional cost)
- ✅ Highly available (built into fabric)

### How This Helps Your Technical Assessment

**When reviewing Azure architectures, you can now**:

1. Identify if NSGs are properly configured
1. Understand the security posture
1. Spot missing network segmentation
1. Recognize over-permissive rules
1. Suggest improvements based on deep understanding

**Questions you can answer**:

- “Why use NSGs at both subnet and NIC level?”
- “How does this scale?”
- “What happens if rules conflict?”
- “Is this configuration following best practices?”

### Connection to Broader Networking

**NSG builds on concepts you learned**:

- IP addressing → sourceAddressPrefix
- Subnetting → Subnet-level NSGs
- Ports → destinationPortRange
- TCP/IP layers → Where NSG operates
- Stateful firewalls → Connection tracking

**Every Azure service uses these fundamentals**:

- Load Balancer (Layer 4 from TCP/IP model)
- Application Gateway (Layer 7 from OSI model)
- VPN Gateway (Routing from Network layer)
- Private Link (IP addressing fundamentals)

-----

## Next Steps

Would you like me to reverse engineer another Azure networking service?

**Good candidates**:

1. **Azure Load Balancer** - How it distributes traffic
1. **Azure Virtual Network** - The foundational networking
1. **Azure VPN Gateway** - How encrypted tunnels work
1. **Azure Application Gateway** - Layer 7 load balancing
1. **Azure Private Link** - Private endpoint technology

Each breakdown will follow the same pattern:

- What you see in portal
- Components and structure
- How it works
- Underlying technology
- TCP/IP mapping
- Physical implementation
- Practical examples

This approach gives you a complete understanding from user interface to bits on the wire!
