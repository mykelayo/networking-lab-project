# Goal 4: STP/RSTP

This goal focuses on mastering Spanning Tree Protocol (STP) and its rapid version (RSTP) to ensure a loop-free Layer 2 topology, predictable forwarding paths, fast convergence, and protection against misconfigurations. We'll verify default behavior, optimize the topology by manually electing root bridges, and apply best-practice features like PortFast, BPDUGuard, and Root Guard.

### Objective
- Verify default STP/RSTP behavior and identify the current root bridge.
- Configure Rapid Per-VLAN Spanning Tree (Rapid-PVST) for per-VLAN instances and faster convergence.
- Manually elect **MLSW1** as the **primary root bridge** for VLANs 10 and 20 (Data & Voice – the most critical/critical traffic).
- Manually elect **MLSW2** as the **primary root bridge** for VLANs 30 and 99 (Wireless & Management – less latency-sensitive).
- Configure the opposite switches as **secondary root** for the other VLANs to provide deterministic failover.
- Apply Root Guard on distribution-to-access links.
- Ensure PortFast and BPDUGuard remain active on access ports.
- Demonstrate redundancy by observing blocked/alternate ports and preparing for HSRP alignment.

This creates a predictable, optimized, and redundant Layer 2 foundation that will perfectly align with HSRP in Goal 12 (active/standby gateways matching root/backup root).

### Why These Configurations?

#### Primary vs Secondary Root Bridge
- **Primary Root**: The switch with the lowest bridge priority for a given VLAN becomes the root bridge. All traffic flows toward it, and it determines the topology.
- **Secondary Root**: A backup root with the next-lowest priority (typically 4096 higher than primary). If the primary fails, the secondary immediately takes over with minimal disruption.
- **Why Per-VLAN Load Balancing?**  
  By splitting primary root responsibility:
  - VLANs 10 & 20 → MLSW1 is root → ASW’s uplink to MLSW1 becomes Root Port (forwarding), uplink to MLSW2 becomes Alternate (blocking).
  - VLANs 30 & 99 → MLSW2 is root → ASW’s uplink to MLSW2 becomes Root Port (forwarding), uplink to MLSW1 becomes Alternate (blocking).
  - **Result**: Traffic is naturally load-balanced across both uplinks while still preventing loops.
- **Alignment with Future HSRP**:  
  We’ll later configure HSRP so that MLSW1 is Active gateway for VLANs 10 & 20, and MLSW2 is Active for VLANs 30 & 99. This ensures optimal Layer 3 forwarding matches the Layer 2 root (traffic doesn’t unnecessarily cross the MLSW1↔MLSW2 EtherChannel).

#### Rapid-PVST Benefits
- Legacy STP convergence: 30–50 seconds (Max Age + 2× Listening + 2× Learning).
- RSTP (integrated into Rapid-PVST): Convergence in seconds (typically <3s for link failures) via proposal/agreement mechanism.
- Per-VLAN instances allow independent topologies and load balancing (unlike MSTP, which is more complex).

#### Protection Features
- **Root Guard**: Prevents a downstream switch from sending superior BPDUs and becoming root (protects against rogue switches or misconfigurations).
- **PortFast + BPDUGuard**: Immediate forwarding on access ports + disables port if a BPDU is received (protects against accidental loops).

### Topology Recap (Layer 2 Redundancy)
- MLSW1 ↔ MLSW2: EtherChannel Po1 (treated as single logical link by STP).
- ASW → MLSW1: Fa0/1 trunk (single link).
- ASW → MLSW2: Fa0/2 trunk (single link).
- This forms a triangle → STP will block **one** ASW uplink per VLAN group.

### Steps in Packet Tracer

#### 1. Verify Default Behavior (Before Optimization)
On **all switches**, run these and capture screenshots:

```
show spanning-tree summary
show spanning-tree vlan 10 
```
![MLSW2 Root Bridge Confirmation](https://github.com/mykelayo/networking-lab-project/blob/main/topology/root-bridge.png)
![MLSW1 spanning-tree vlan 10](https://github.com/mykelayo/networking-lab-project/blob/main/topology/spanning-tree-vlan-10-after.png)
![ASW Designated/Alternate](https://github.com/mykelayo/networking-lab-project/blob/main/topology/ASW-root-alt.png)

**Observation**:
- Root bridge is **MLSW2** 
- Priorities are all 32778.
- One ASW uplink is Designated (forwarding), the other Alternate (blocking).
- Convergence would be slow if using legacy STP.

#### 2. Enable Rapid-PVST Globally
On **MLSW1, MLSW2, and ASW**:
```
conf t
spanning-tree mode rapid-pvst
end
```

#### 3. Manually Elect Primary and Secondary Root Bridges

**On MLSW1** (Primary for Data & Voice, Secondary for Wireless & Mgmt):
```
conf t
spanning-tree vlan 10,20 root primary
spanning-tree vlan 30,99 root secondary
end
```

**On MLSW2** (Primary for Wireless & Mgmt, Secondary for Data & Voice):
```
conf t
spanning-tree vlan 30,99 root primary
spanning-tree vlan 10,20 root secondary
end
```

**What Happens Under the Hood**:
- `root primary` → Sets priority to 24576 (or lower if needed).
- `root secondary` → Sets priority to 28672.
- ASW remains at default 32768 → Cannot become root.

#### 4. Apply Root Guard on Distribution Links
Protect the core from inferior switches claiming superiority.

On **MLSW1**:
```
 conf t
 interface FastEthernet0/4
 spanning-tree guard root
 end
```

On **MLSW2**:
```
 conf t
 interface FastEthernet0/4
 spanning-tree guard root
 end
```

#### 5. Apply PortFast and BPDUGuard on Access Ports

On **ASW**:
```
 conf t
 interface range FastEthernet0/3 - 4
 spanning-tree portfast
 spanning-tree bpduguard enable
 end
```

#### 6. Save All Configurations
On all switches:
```
end
write memory
```

### Verification Commands (After Optimization)

1. **Root Bridge Election**
   On **MLSW1**:
   ```
   show running-config | section spanning-tree
   ```
   On **MLSW2**:
   → Opposite for VLANs 30,99.
   ![MLSW1 spanning-tree cli](https://github.com/mykelayo/networking-lab-project/blob/main/topology/cli-span-tree-veri.png)

2. **EtherChannel Treatment**
   ```
   show spanning-tree vlan 10
   ```
   ![Port-Channel State](https://github.com/mykelayo/networking-lab-project/blob/main/topology/ether-treatment.png)
   → Po1 between MLSW1↔MLSW2 is Forwarding (single logical link).

### What We Achieved
- Deterministic root bridge election with **per-VLAN load balancing**.
- Rapid convergence enabled via Rapid-PVST.
- Protection against rogue root claims (Root Guard) and access-layer loops (BPDUGuard).
- Traffic naturally distributed across both ASW uplinks.
- Layer 2 topology now perfectly aligned for HSRP (MLSW1 active for VLANs 10/20, MLSW2 active for 30/99).
- Demonstrated real world design principles.