# 100 - Azure Network Security Group (NSG)

I’ll create a comprehensive “reverse engineering” breakdown of Azure NSGs, showing how they work from the portal level all the way down to the TCP/IP fundamentals!​​​​​​​​​​​​​​​​

Perfect! I’ve created a comprehensive “reverse engineering” breakdown of Azure NSGs, from the portal all the way down to the TCP/IP packet level!

[View your Azure NSG Reverse Engineering Guide](computer:///mnt/user-data/outputs/Azure-NSG-Reverse-Engineering.md)

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

When Team Rockstars shows you an Azure architecture, you can now:

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

## 
