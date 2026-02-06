# 🔥 Network & SysAdmin Cheat Sheet (Compact)

**Vars:** `<<IP>>`, `<<MASK>>`, `<<GW>>`, `<<AS>>`, `<<PASS>>`

---

## 1. 🟢 EcoRouter (Cisco-like)

```bash
en
conf t
host <<HOSTNAME>>
dom <<DOMAIN>>

! Interfaces
int gi0/0
 desc WAN
 ip addr <<WAN_IP>> <<WAN_MASK>>
 no shut
int gi0/1
 desc LAN
 ip addr <<LAN_IP>> <<LAN_MASK>>
 no shut

! BGP
router bgp <<MY_AS>>
 bgp r-id <<WAN_IP>>
 nei <<ISP_GW>> remote-as <<ISP_AS>>
 addr ipv4
  nei <<ISP_GW>> act
  net <<WAN_NET>> mask <<WAN_MASK>>
  exit
 nei <<ISP_GW>> default-originate

! Tunnel (GRE)
int Tun1
 ip addr <<TUN_IP>> 255.255.255.252
 tun source <<WAN_IP>>
 tun dest <<REMOTE_WAN_IP>>
 tun mode gre ip
 ip ospf message-digest-key 1 md5 <<PASS>>

! OSPF
router ospf 1
 r-id 1.1.1.1
 pass def
 no pass Tun1
 no pass gi0/1
 net <<TUN_NET>> 0.0.0.3 area 0
 net <<LAN_NET>> <<WILDCARD>> area 0
 area 0 auth message-digest

Interface,File: options,File: ipv4address
bond0,TYPE=bondbond-mode 1bond-miimon 100,BOOTPROTO=static
vlan300 (Mgmt),TYPE=vlanVID=300HOST=bond0,<<IP>>/24
bond0.100,TYPE=vlanVID=100HOST=bond0,-
ens6.100,TYPE=vlanVID=100HOST=ens6,-
br100,"TYPE=briPORTS=""bond0.100 ens6.100""",-

systemctl restart network

