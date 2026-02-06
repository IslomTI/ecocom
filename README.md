Конечно, вот максимально компактная и структурированная версия для `README.md`.

Я использовал следующие методы для улучшения:

1. **Компактность:** Конфигурационные файлы записаны в формате `Путь -> Содержимое` (как ты просил), чтобы не писать громоздкие `echo`.
2. **Синтаксис:** Использовал тег `shell` (наиболее универсальная подсветка на GitHub) для всех блоков.
3. **Без "воды":** Убрал текст между блоками, все пояснения внутри кода как комментарии (`#`).
4. **Цвет:** В Markdown на GitHub нельзя принудительно покрасить слово в синий цвет внутри блока кода, но я использовал `shell`, где переменные в верхнем регистре и конструкции `<<...>>` обычно выделяются лучше всего.

Скопируй код ниже в свой файл.

```markdown
# 🚀 Network & System Admin Cheat Sheet (Compact)

> **Legend:** `<<...>>` = Переменные из задания.

---

## 1. 🟢 EcoRouter

```shell
enable
conf t
host <<HOSTNAME>>
dom <<DOMAIN>>

# --- Interfaces ---
int gi0/0
 desc WAN
 ip addr <<WAN_IP>> <<WAN_MASK>>
 no shut
int gi0/1
 desc LAN
 ip addr <<LAN_IP>> <<LAN_MASK>>
 no shut

# --- BGP ---
router bgp <<MY_AS>>
 bgp r-id <<WAN_IP>>
 nei <<ISP_GW>> remote-as <<ISP_AS>>
 addr ipv4
  nei <<ISP_GW>> act
  # Анонс СВОЕЙ сети (совпадение маски!)
  net <<WAN_NET>> mask <<WAN_MASK>>
  exit
 nei <<ISP_GW>> default-originate

# --- Tunnel (GRE) ---
int Tun1
 ip addr <<TUN_IP>> 255.255.255.252
 tun source <<WAN_IP>>
 tun dest <<REMOTE_WAN_IP>>
 tun mode gre ip
 ip ospf message-digest-key 1 md5 <<PASS>>

# --- OSPF ---
router ospf 1
 r-id 1.1.1.1
 pass def
 no pass Tun1
 no pass gi0/1
 # Wildcard маски (/30 -> 0.0.0.3)
 net <<TUN_NET>> 0.0.0.3 area 0
 net <<LAN_NET>> <<WILDCARD>> area 0
 area 0 auth message-digest

```

## 2. 🟢 EcoRouter (RADIUS Client)

```shell
enable
conf t
aaa new-model
radius-server host 192.168.100.10 key radius_secret
aaa authentication login default group radius local
username admin privilege 15 secret P@ssw0rdLocal

line console 0
 login authentication default
exit
line vty 0 4
 login authentication default
 trans in ssh
exit

```

---

## 3. 🐧 Alt Linux Network

```shell
hostnamectl set-hostname <<HOSTNAME>>

# === Switch Config (Bond + VLANs) ===
# /etc/net/ifaces/bond0/options -> TYPE=bond bond-mode 1 bond-miimon 100
# /etc/net/ifaces/bond0/ports -> ens4 ens5
# /etc/net/ifaces/bond0/ipv4address -> BOOTPROTO=static

# Clean physical ports
rm -rf /etc/net/ifaces/ens4 /etc/net/ifaces/ens5

# VLAN 300 (L3 Mgmt)
# /etc/net/ifaces/vlan300/options -> TYPE=vlan VID=300 HOST=bond0
# /etc/net/ifaces/vlan300/ipv4address -> <<SWITCH_IP>>/24
# /etc/net/ifaces/vlan300/ipv4route -> default via <<GW_IP>>

# VLAN 100 (L2 Bridge)
# /etc/net/ifaces/bond0.100/options -> TYPE=vlan VID=100 HOST=bond0
# /etc/net/ifaces/ens6.100/options -> TYPE=vlan VID=100 HOST=ens6
# /etc/net/ifaces/br100/options -> TYPE=bri PORTS="bond0.100 ens6.100"

systemctl restart network
ip a

```

---

## 4. 💾 Storage (iSCSI & NFS)

```shell
# === SRV2 (Target) ===
# Network: ens18 -> <<IP>>/24, GW: <<GW>>
apt-get install targetcli
systemctl enable --now target

targetcli
# /backstores/block create name=d1 dev=/dev/sdb
# /iscsi create <<IQN_TARGET>>
# /iscsi/<<IQN_TARGET>>/tpg1/acls create <<IQN_INITIATOR>>
# /iscsi/<<IQN_TARGET>>/tpg1/luns create /backstores/block/d1
exit

# === SRV1 (Initiator) ===
apt-get install open-iscsi lvm2 nfs-utils

# Connect
# /etc/iscsi/initiatorname.iscsi -> InitiatorName=<<IQN_INITIATOR>>
systemctl restart iscsid
iscsiadm -m disc -t st -p <<SRV2_IP>>
iscsiadm -m node -l

# LVM Setup
pvcreate /dev/sdb
vgcreate VG /dev/sdb
lvcreate -l 100%FREE -n DATA VG
mkfs.xfs /dev/VG/DATA

# Mount
mkdir -p /opt/data
# blkid /dev/VG/DATA -> UUID
# /etc/fstab -> UUID="<<UUID>>" /opt/data xfs _netdev 0 0
mount -a

# NFS Export
# /etc/exports -> /opt/data <<NETWORK>>/24(rw,sync,no_root_squash)
exportfs -ra
systemctl enable --now nfs-server

```

---

## 5. 🛠️ Services (DNS, RADIUS, CA)

```shell
# === RADIUS (srv1) ===
apt-get install freeradius freeradius-utils
# /etc/raddb/clients.conf -> client <<NAME>> { ipaddr=<<IP>> secret=<<PASS>> }
# /etc/raddb/users -> <<USER>> Cleartext-Password := "<<PASS>>" Service-Type = Administrative-User
systemctl enable --now radiusd

# === DNS (srv1) ===
apt-get install bind bind-utils
# /etc/bind/options.conf -> forwarders { 100.100.100.100; };
# /etc/bind/local.conf -> zone "<<DOMAIN>>" { type master; file "/var/lib/bind/db.dom"; };

# Zone File (/var/lib/bind/db.dom)
# @ IN SOA srv1. root. ( ... )
# @ IN NS srv1.
# @ IN A <<SRV1_IP>>
# monitoring IN CNAME srv1
systemctl enable --now bind

# === CA (srv1) ===
mkdir -p /var/ca/{certs,crl,newcerts,private}
touch /var/ca/index.txt; echo 1000 > /var/ca/serial
cp /etc/ssl/openssl.cnf /var/ca/openssl.cnf
# Edit openssl.cnf: dir=/var/ca, days=1825, CN=ssa2026

cd /var/ca
openssl genrsa -out private/ca.key 4096
openssl req -new -x509 -key private/ca.key -out ca.crt -config openssl.cnf
cp ca.crt /usr/share/ca-certificates/ssa2026.crt
update-ca-trust

```

---

## 6. 🏢 Active Directory (Samba)

```shell
# === DC-A ===
hostnamectl set-hostname dc-a.office.ssa2026.region
apt-get install samba-dc bind bind-utils
rm -f /etc/samba/smb.conf

# Provision
samba-tool domain provision --realm=<<REALM>> --domain=<<DOM>> --adminpass=<<PASS>> --server-role=dc --dns-backend=BIND9_DLZ

# Bind Integration
# /etc/bind/named.conf -> include "/var/lib/samba/bind-dns/named.conf";
# Rights -> chgrp named /var/lib/samba/private/dns.keytab

systemctl disable smb nmb
systemctl enable --now samba-ad-dc
systemctl restart bind

# Users & Groups
samba-tool ou create "OU=ofadmins"
samba-tool user create <<USER>> <<PASS>> --userou="OU=ofadmins"

```

---

## 7. 📊 Zabbix & SIP

```shell
# === Zabbix DB (srv2) ===
# pg_hba.conf -> host all all 0.0.0.0/0 scram-sha-256
# SQL: create user zabbix_user password '...'; create database zabbix owner zabbix_user;

# === Zabbix Server (srv1) ===
apt-get install zabbix-server-pgsql zabbix-web-apache-pgsql
# Import DB: zcat ... | psql -h <<SRV2>> -U zabbix_user -d zabbix
# zabbix_server.conf -> DBHost=<<SRV2>>, DBPassword=...
# ssl.conf -> SSLCertificateFile /var/ca/certs/srv1.crt

# === Asterisk (SIP) ===
# Browser: http://<<SIP_IP>>
# Settings -> SIP Settings -> PJSIP (5160) / Chan_SIP (5060)
# Extensions -> Create 1001, 1002...

```

```

```
