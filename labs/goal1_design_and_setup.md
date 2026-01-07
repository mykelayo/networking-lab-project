## Goal 1: Design Topology and Basic Setup
Now, let's dive into the first lab. This includes detailed explanations, steps in PT, and basic initial configs (like hostnames, no shutdown on interfaces). We'll keep it simple, no IPs yet (that's Goal 2). 

### labs/goal1_design_and_setup.md

#### Objective
In this initial lab, we'll design the network topology, add devices in Packet Tracer, cable them according to the physical diagram, and perform basic setup. This ensures a solid foundation before subnetting and configurations. The focus is on physical connectivity and minimal CLI tweaks to get devices ready (e.g., hostnames, interface up). We'll verify basic reachability isn't needed yet, that's later.

#### Why This Topology?
- **Redundancy**: Dual MLSWs with HSRP (later) and EtherChannels prevent single points of failure.
- **Scalability**: Supports VLANs across switches, wireless management via WLC, and voice via CME.
- **Feature Coverage**: Allows router-on-a-stick (subinterfaces on CME-R), SVI routing (on MLSWs), and multi-router dynamics.

### Logical Topology
![Logical Topology](topology/logical-topology.png)

### Physical Topology
![Physical Topology on PT](topology/physical-topology.png)

#### Steps in Packet Tracer
1. **Open Packet Tracer and Add Devices**:
   - Go to Devices > Switches: Add two 3560 (MLSW1, MLSW2) and one 2960 (ASW).
   - Devices > Routers: Add two 2811 (CME-R), 2911 (Edge-R) and one 1941 (ISP-R).
   - Devices > Wireless: Add WLC-3504 (WLC) and LAP-PT (LWAP).
   - End Devices: Add PC1, PC2, IP-Phone1, IP-Phone2, Laptop, Server.
   - Tip: Label each device by right-clicking > Inspect > Label.

2. **Cable the Devices**:
   - MLSW1 Fa0/1 to Edge-R Gi0/0.
   - MLSW2 Fa0/1 to Edge-R Gi0/1 (For redundancy).
   - MLSW1 Fa0/2-3 to MLSW2 Fa0/2-3 (EtherChannel).
   - MLSW1 Fa0/5 to WLC Ethernet1 (Management port).
   - ASW Fa0/1 to MLSW1 Fa0/4 (Trunk).
   - ASW Fa0/2 to MLSW2 Fa0/4 (Trunk).
   - ASW Fa0/3 to IP-Phone1/PC1 (Phone to PC for daisy-chain).
   - ASW Fa0/4 to IP-Phone2/PC2.
   - ASW Fa0/5 to LWAP.
   - ASW Fa0/6 to CME-R Gi0/0 (Trunk for subinterfaces).
   - ASW Fa0/7 to Server.
   - Edge-R Serial0/3/0 to ISP-R Serial0/1/0.
   - Power on all devices (green light).

3. **Basic Setup on Each Device** (Enter CLI via Console):
   - For all devices: Enter enable mode (`en`), then config mode (`conf t`).
   - Set hostnames:
     - MLSW1: `hostname MLSW1`
     - MLSW2: `hostname MLSW2`
     - ASW: `hostname ASW`
     - CME-R: `hostname CME-R`
     - Edge-R: `hostname Edge-R`
     - ISP-R: `hostname ISP-R`
     - WLC: Access via console, set name if possible (WLC CLI is limited in PT).
   - Bring up interfaces: On each switch/router, `int range fa0/1 - 24, gi0/0 - 1`, `no shut`.
   - On WLC: Basic setup wizard if prompted, but skip advanced for now.
   - Set clock timezone: `clock timezone WAT 1` (personalize to your zone).
   - Enable secret: `enable secret cisco`.
   - Enable username and password: `username admin privilege 15 secret cisco`
   - Service password-encryption: `service password-encryption`.
   - Configure console: `line console 0`, `login local`, `exec-timeout 30`, `logging synchronous`
   - MOTD Banner: `banner motd # Welcome to My Networking Lab Project #`.
   - Save configs: `end`, `wr`.

4. **Potential Issues and Tips**:
   - PT Crashing: Add devices one by one, save often. Use logical workspace view.
   - WLC in PT: Limited simulation; focus on basics like WLAN creation later.
   - Why Basic Setup First? Prevents confusion in later labs and demonstrates initial hardening.