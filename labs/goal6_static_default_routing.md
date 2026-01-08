# Goal 6: Static and Default Routing

We now extend Layer 3 connectivity beyond the campus network by configuring **static routes** and **default routes** to simulate Internet access through Edge-R and ISP-R. This goal will introduce external reachability, demonstrates route summarization considerations, floating static routes for redundancy, and prepares the network for dynamic routing protocols in Goal 7.

At the end of this goal:
- Internal devices can reach the simulated "Internet" (ISP-R loopback).
- ISP-R can reach internal subnets (for return traffic testing).
- Edge-R uses dual static routes to MLSWs for redundancy.
- We verify full end-to-end connectivity from campus hosts to the external loopback.

### Objective
- Configure static routes on Edge-R toward internal campus subnets via MLSW1 and MLSW2.
- Implement **floating static routes** on Edge-R for redundancy (primary via MLSW1, backup via MLSW2 or vice versa).
- Configure a **default route** on both MLSW1 and MLSW2 pointing to Edge-R.
- Configure a **default route** (or static summary) on Edge-R pointing toward ISP-R.
- Configure static routes on ISP-R back to internal networks (for bidirectional testing).
- Verify routing tables, traceroute behavior, and redundancy failover.
- Document route preferences and prepare for dynamic routing replacement (OSPF/EIGRP).

### Why Static and Default Routing?

#### Key Concepts Demonstrated
- **Default Route (0.0.0.0/0)**: The "gateway of last resort", this is used when no specific route matches. Essential for Internet access.
- **Static Routes**: Manually configured, administratively controlled paths. Useful for stub networks, small sites, or as backup.
- **Floating Static Routes**: Same destination with higher administrative distance (AD) — acts as backup if primary fails. AD default: static = 1, floating > 1.
- **Redundancy Without Dynamic Protocols**: Shows how to achieve failover using multiple static routes before introducing OSPF/EIGRP.
- **Route Summarization**: Internal campus is 10.0.0.0/16 can be summarized toward ISP-R.

#### Design Decisions
- **MLSWs → Edge-R**: Default route (simple, scalable for campus).
- **Edge-R → MLSWs**: Two static routes to internal subnets (load-share or primary/backup).
- **Edge-R → ISP-R**: Default route (Edge-R is border).
- **ISP-R → Internal**: Summary static route (10.0.0.0/16) for simplicity.
- **Alignment with Previous Goals**:
  - Uses P2P links from Goal 2 (10.0.1.0/30 and 10.0.1.4/30).
  - Prepares for HSRP (default route points to physical Edge-R IPs; later can use loopback).
  - Sets stage for dynamic routing convergence testing.

### Routing Plan Summary Table

| Device     | Destination              | Next-Hop / Exit Interface       | Administrative Distance | Purpose / Notes                     |
|------------|--------------------------|---------------------------------|--------------------------|-------------------------------------|
| MLSW1      | 0.0.0.0/0                | 10.0.1.2 (Edge-R Gi0/0)         | 1                        | Default → Internet                  |
| MLSW2      | 0.0.0.0/0                | 10.0.1.6 (Edge-R Gi0/1)         | 1                        | Default → Internet                  |
| Edge-R     | 10.0.0.0/16              | 10.0.1.1 (MLSW1)                | 1                        | Primary internal summary            |
| Edge-R     | 10.0.0.0/16              | 10.0.1.5 (MLSW2)                | 10                       | Floating backup                     |
| Edge-R     | 0.0.0.0/0                | 203.0.113.2 (ISP-R)             | 1                        | Default → Internet                  |
| ISP-R      | 10.0.0.0/16              | 203.0.113.1 (Edge-R)            | 1                        | Return traffic to campus            |
| ISP-R      | 0.0.0.0/0                | (optional loopback self)        | —                        | Not needed for simulation           |

> Note: We use a **summary route** 10.0.0.0/16 on Edge-R and ISP-R because all internal VLANs and P2P links fall within this range.

### Steps in Packet Tracer

#### 1. Configure Default Routes on MLSW1 and MLSW2
On **MLSW1**:
```
conf t
ip route 0.0.0.0 0.0.0.0 10.0.1.2
end
write memory
```

On **MLSW2**:
```
conf t
ip route 0.0.0.0 0.0.0.0 10.0.1.6
end
write memory
```

#### 2. Configure Static Routes on Edge-R (with Floating Backup)
On **Edge-R**:
```
conf t
! Primary route via MLSW1
ip route 10.0.0.0 255.255.0.0 10.0.1.1

! Floating backup via MLSW2 (AD 10)
ip route 10.0.0.0 255.255.0.0 10.0.1.5 10

! Default toward ISP
ip route 0.0.0.0 0.0.0.0 203.0.113.2
description Default to ISP-R
end
write memory
```

#### 3. Configure Return Route on ISP-R
On **ISP-R**:
```
conf t
ip route 10.0.0.0 255.255.0.0 203.0.113.1
end
write memory
```

### Comprehensive Verification

#### 1. Routing Tables
On **MLSW1**:
```
show ip route
```

![MLSW1 Routing Table with Default](https://github.com/mykelayo/networking-lab-project/blob/main/topology/mlsw1-route-default.png)

On **Edge-R**:
```
show ip route static
```

![Edge-R Static Routes](https://github.com/mykelayo/networking-lab-project/blob/main/topology/edge-r-static-floating.png)

On **ISP-R**:
```
show ip route
```

![ISP-R Static Routes](https://github.com/mykelayo/networking-lab-project/blob/main/topology/isp-r-static.png)

#### 2. End-to-End Connectivity Tests
From **PC1 (VLAN 10)**:
```
ping 5.5.5.5
```

![PC1 to ISP Loopback Ping](https://github.com/mykelayo/networking-lab-project/blob/main/topology/pc1-to-isp-l0.png)

```
tracert 5.5.5.5
```

![PC1 Traceroute to ISP](https://github.com/mykelayo/networking-lab-project/blob/main/topology/pc1-tracert-isp.png)

From **Server (VLAN 99)**:
```
ping 5.5.5.5
```

![SRV to ISP Ping](https://github.com/mykelayo/networking-lab-project/blob/main/topology/srv-to-isp-l0.png)

From **ISP-R**:
```
ping 10.0.10.10
```

![ISP to PC1 Ping](https://github.com/mykelayo/networking-lab-project/blob/main/topology/isp-to-pc1.png)

#### 3. Redundancy Test (Floating Static)
On **Edge-R**, temporarily shut primary link:
```
conf t
interface Gi0/0
 shutdown
end
```

Then:
```
show ip route 10.0.0.0
```

![Edge-R Redundancy Test](https://github.com/mykelayo/networking-lab-project/blob/main/topology/edge-r-redundancy-test.png)

From Edge-R: `ping 10.0.10.10` 

![Edge-R to SRV ping backup test](https://github.com/mykelayo/networking-lab-project/blob/main/topology/edge-to-srv-backup-test.png)

Bring interface back:

```
interface Gi0/0
 no shutdown
```

→ Primary route returns (lower AD).

#### Static Route Redundancy Behavior
In our setup, PC1 in VLAN 10 connects through the access switch to MLSW1 and MLSW2. MLSW1 owns VLANs 10 and 20, while MLSW2 owns VLANs 30 and 19. The Edge Router has static routes to both MLSWs, but when we shut down the interface to MLSW1, traffic can still reach VLANs on MLSW2, yet return traffic to PC1 fails because MLSW1 is the default gateway for VLAN 10. This shows that with static routes, redundancy only works if both MLSWs have routes for all VLANs, or if a dynamic routing protocol is used. Spanning Tree only controls Layer 2 paths and doesn’t handle Layer 3 routing.

![Edge-R Floating Route Active](https://github.com/mykelayo/networking-lab-project/blob/main/topology/edge-r-floating-active.png)

### What We Achieved
- **Full external connectivity** from campus to simulated Internet.
- **Redundant static routing** using floating routes on Edge-R.
- **Default routing** implemented on distribution layer.
- **Bidirectional reachability** verified with pings and traceroute.
- Demonstrated **route preference**, **summarization**, and **failover**.
- Network now has complete Layer 3 reachability — internal and external.
- Perfect setup for **dynamic routing** introduction (Goal 7: OSPF/EIGRP/RIP).