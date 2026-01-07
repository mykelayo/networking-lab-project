# Goal 3: VLANs and Trunks

This goal focuses on creating VLANs, assigning access ports, configuring trunk links (including EtherChannels), and ensuring proper VLAN propagation across the network. We'll also configure voice VLANs for IP phones and prepare trunks for the Wireless LAN Controller (WLC) and Access Point (LWAP).

### Objective
- Create and name VLANs on all switches.
- Configure access ports for end devices (PCs, phones, server).
- Configure voice VLANs on phone ports for dual-VLAN support (data + voice).
- Configure trunk ports between switches
- Configure EtherChannel between the two multilayer switches (MLSW1 ↔ MLSW2)
- Configure trunks to CME-R (for subinterfaces) and to WLC.
- Allow only necessary VLANs on trunks (security best practice).
- Set native VLAN consistently.
- Verify VLAN database consistency, trunk status, and connectivity within VLANs.

This will bring up the SVIs from Goal 2 and allow hosts in the same VLAN to ping each other.

### VLAN Design Recap
From Goal 2:
| VLAN | Name        | Subnet          | Purpose                  |
|------|-------------|-----------------|--------------------------|
| 10   | Data        | 10.0.10.0/24    | PCs, general data        |
| 20   | Voice       | 10.0.20.0/24    | IP Phones (CME)          |
| 30   | Wireless    | 10.0.30.0/24    | Wireless clients(via WLC)|
| 99   | Management  | 10.0.99.0/24    | Switch/WLC management    |

### Why These Configurations?
- **Trunks with Allowed VLANs**: Restricts unnecessary VLAN traffic (security + efficiency).
- **EtherChannel**: Provides redundancy and doubles bandwidth between MLSWs.
- **Voice VLAN**: Allows a single cable to carry both PC data (VLAN 10) and phone voice (VLAN 20).
- **Native VLAN 99**: Management VLAN as native avoids CDP/LLDP warnings and keeps untagged management traffic separate.
- **Switchport nonegotiate**: Disables DTP to prevent trunk negotiation attacks.

### Steps in Packet Tracer

#### 1. Create VLANs on All Switches (MLSW1, MLSW2, ASW)
On **each switch** (MLSW1, MLSW2, ASW):
```
conf
vlan 10
 name Data
vlan 20
 name Voice
vlan 30
 name Wireless
vlan 99
 name Management
exit
```

#### 2. Configure Access Ports on ASW
On **ASW**:
```
 conf t
! PC1 and PC2 (behind phones) - Data VLAN
 interface FastEthernet0/3
 description PC1-behind-Phone1
 switchport mode access
 switchport access vlan 10

 interface FastEthernet0/4
 description PC2-behind-Phone2
 switchport mode access
 switchport access vlan 10

! IP Phones - Voice VLAN
 interface range FastEthernet0/3 - 4
 switchport voice vlan 20

! LWAP (Lightweight AP) AP uses management vlan
 interface FastEthernet0/5
 description LWAP
 switchport mode access
 switchport access vlan 99

! Server (DNS/Syslog later)
 interface FastEthernet0/7
 description Server
 switchport mode access
 switchport access vlan 99
```

#### 3. Configure Trunk Ports with EtherChannel (ASW to MLSW1 & MLSW2)
We'll bundle two ports per link for redundancy.

On **ASW**:
```
 conf t
 interface FastEthernet0/1
 description Trunk to MLSW1 Fa0/4
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,99
 switchport nonegotiate

 interface FastEthernet0/2
 description Trunk to MLSW2 Fa0/4
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,99
 switchport nonegotiate
```

On **MLSW1** (for link to ASW):
```
 conf t
 interface FastEthernet0/4
 description Trunk to ASW Fa0/1
 switchport
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,99
 switchport nonegotiate
```

On **MLSW1** (for link to MLSW2):
```
 conf t
 interface range FastEthernet0/2-3
 channel-group 1 mode active
 exit

 interface Port-channel 1
 description Trunk to MLSW2
 switchport
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,99
 switchport nonegotiate
```

On **MLSW2** (for link to ASW):
```
 conf t
 interface FastEthernet0/4
 description Trunk to ASW Fa0/2
 switchport
 switchport trunk encapsulation dot1q 
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,99
 switchport nonegotiate
```

On **MLSW2** (for link to MLSW1):
```
 conf t
 interface range FastEthernet0/2-3
 channel-group 1 mode active
 exit

 interface Port-channel 1
 description Trunk to MLSW1
 switchport
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,99
 switchport nonegotiate
```

#### 4. Trunk to CME-R (Router-on-a-Stick)
On **ASW**:
```
 conf t
 interface FastEthernet0/6
 description Trunk to CME-R Fa0/0
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 20,99 
 switchport nonegotiate
```

No changes needed on CME-R physical interface, we have already subinterfaces configured in Goal 2

#### 5. Trunk to WLC
On **MLSW1** (WLC is connected here):
```
 conf t
 interface FastEthernet0/5
 description Trunk to WLC
 switchport
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 99,30
 switchport nonegotiate
```

#### 6. Shutdown all unused port (All Switches)
On **MLSW1**:
```
 conf t
 interface range FastEthernet0/6 - 24, GigabitEthernet0/1 - 2
 description Unused-Port-Shutdown
 shutdown
 exit
```

On **MLSW2**:
```
 conf t                    
 interface range FastEthernet0/5 - 24, GigabitEthernet0/1 - 2
 description Unused-Port-Shutdown
 shutdown
 exit
```

On **ASW**:
```
 conf t
 interface range FastEthernet0/8 - 24, GigabitEthernet0/1 - 2
 description Unused-Port-Shutdown
 shutdown
 exit
```

#### 7. Save Configurations
On all devices:
```
end
write memory
```

### Verification Commands
On any switch:
- `show vlan brief` → All VLANs present and correct ports assigned  
![VLANs config](https://github.com/mykelayo/networking-lab-project/blob/main/topology/vlans-brief.png)  
- `show interfaces trunk` → Trunks up, native VLAN 99, allowed lists correct  
![Interface trunk status](https://github.com/mykelayo/networking-lab-project/blob/main/topology/trunks-status.png)  
- `show etherchannel summary` → Port-channels 1 and 2 in (SU) state (bundled)  
![Etherchannel between MLSW1 and MLSW2](https://github.com/mykelayo/networking-lab-project/blob/main/topology/ether-sum.png)  
- `show interfaces switchport` → Confirm access/trunk modes, voice VLANs  
![Interfaces Switchport Trunk for ASW Fa0/1](https://github.com/mykelayo/networking-lab-project/blob/main/topology/ASW-to-MLSW-int-trunk.png)  
![Interfaces Switchport Access for ASW Fa0/3](https://github.com/mykelayo/networking-lab-project/blob/main/topology/ASW-to-IP-Phone-PC1-int-sw.png)  
- Once WLC configured later, wireless will join VLAN 30.
From MLSW1:
- `show ip interface brief` → VLAN 10,20,30,99 interfaces now **up/up**
![MLSW1 IP interface](https://github.com/mykelayo/networking-lab-project/blob/main/topology/MLSW1-ip-int-br.png)

### Potential Issues & Tips
- **EtherChannel not forming?** Ensure same port groups, mode (active/active for LACP), and cabling.
- **Trunk not allowing VLANs?** Double-check `allowed vlan` lists.
- **SVIs still down?** Ensure at least one active port in the VLAN (e.g., connect a PC).
- **WLC trunk?** WLC in PT needs management in VLAN 99; client traffic tagged in VLAN 30.

### What We Achieved
- All devices now communicate within their respective VLANs (intra-VLAN connectivity working).
- Redundant trunk paths established between core switches (MLSW1 ↔ MLSW2 via EtherChannel).
- Voice and data traffic separated on phone ports using voice VLAN.
- Trunk security applied (allowed VLANs, no DTP negotiation, native VLAN match).
- Foundation laid for inter-VLAN routing (SVIs now up), HSRP, and wireless.