Вот исправленная, структурированная и отформатированная версия для GitHub (`README.md`).

Я исправил синтаксические ошибки (например, лишние скобки, неправильные команды `apt-get`, логику BGP) и оформил всё в блоки кода для удобного копирования.

---

# 🚀 Network & System Administration Cheat Sheet

Шпаргалка для настройки оборудования и сервисов (EcoRouter, Alt Linux, Samba, Zabbix).
**Легенда:** Всё, что в `<< >>` — это переменные, которые нужно менять под ситуацию.

---

## 🌐 1. EcoRouter (Network)

```bash
enable
configure terminal

! 1. Базовая настройка
hostname <<HOSTNAME>>
domain-name <<DOMAIN_NAME>>

! 2. Интерфейсы
! Физический интерфейс
interface gigabitethernet <<INT_NUM>>
 description <<DESC>>
 ip address <<IP_ADDRESS>> <<MASK>>
 no shutdown
exit

! Саб-интерфейс (VLAN)
interface gigabitethernet <<INT_NUM>>.<<VLAN_ID>>
 encapsulation dot1Q <<VLAN_ID>>
 ip address <<IP_ADDRESS>> <<MASK>>
 no shutdown
exit

! 3. BGP (Связь с провайдером)
router bgp <<MY_AS>>
 bgp router-id <<MY_ROUTER_ID_IP>>
 neighbor <<ISP_IP>> remote-as <<ISP_AS>>
 neighbor <<ISP_IP>> description ISP
 address-family ipv4
  neighbor <<ISP_IP>> activate
  ! Анонсируем СВОЮ сеть (не шлюз!), маска обычная
  network <<MY_EXTERNAL_NET>> mask <<MASK>>
  ! Если нужно отдать дефолт соседу (не провайдеру, а внутрь):
  ! neighbor <<NEIGHBOR_IP>> default-originate
 exit
exit

! 4. GRE Туннель
interface Tunnel1
 ip address <<TUNNEL_IP>> 255.255.255.252
 tunnel source <<MY_EXTERNAL_IP>>
 tunnel destination <<REMOTE_EXTERNAL_IP>>
 tunnel mode gre ip
 ip ospf message-digest-key 1 md5 <<OSPF_PASS>>
exit

! 5. OSPF (Внутренняя маршрутизация)
router ospf 1
 router-id <<ROUTER_ID_IP>>
 passive-interface default
 no passive-interface Tunnel1
 ! Включаем интерфейсы, смотрящие в локалку
 no passive-interface gigabitethernet <<LAN_INT>>
 area 0 authentication message-digest
 ! Сети с обратной маской (Wildcard): /30 -> 0.0.0.3, /24 -> 0.0.0.255
 network <<TUNNEL_NET>> 0.0.0.3 area 0
 network <<LAN_NET>> <<WILDCARD>> area 0
exit
write

```

---

## 🔐 2. EcoRouter (RADIUS Client)

```bash
enable
conf t

aaa new-model
radius-server host <<RADIUS_SERVER_IP>> key <<RADIUS_SECRET>>
aaa authentication login default group radius local

username admin privilege 15 secret <<LOCAL_PASS>>

line console 0
 login authentication default
exit

line vty 0 4
 login authentication default
 transport input ssh
exit
write

```

---

## 🐧 3. Alt Linux Network (`etcnet`)

> **Путь:** `/etc/net/ifaces/`
> **Применение:** `systemctl restart network` и `ip a`

### LACP Aggregation (Bonding)

```bash
# 1. Создаем Bond0
mkdir -p /etc/net/ifaces/bond0
echo "TYPE=bond" > /etc/net/ifaces/bond0/options
echo "bond-mode 1" >> /etc/net/ifaces/bond0/options
echo "bond-miimon 100" >> /etc/net/ifaces/bond0/options
echo "BOOTPROTO=static" > /etc/net/ifaces/bond0/ipv4address

# Добавляем порты в bond0
echo "ens4" > /etc/net/ifaces/bond0/ports
echo "ens5" >> /etc/net/ifaces/bond0/ports

# Чистим физические порты (чтобы не мешали)
mkdir -p /etc/net/ifaces/ens4 /etc/net/ifaces/ens5
echo "TYPE=eth" > /etc/net/ifaces/ens4/options
echo "TYPE=eth" > /etc/net/ifaces/ens5/options

```

### VLAN (L3 Interface - Management)

```bash
# Интерфейс с IP-адресом (шлюз для свитча)
mkdir -p /etc/net/ifaces/vlan<<VLAN_ID>>
echo "TYPE=vlan" > /etc/net/ifaces/vlan<<VLAN_ID>>/options
echo "VID=<<VLAN_ID>>" >> /etc/net/ifaces/vlan<<VLAN_ID>>/options
echo "HOST=bond0" >> /etc/net/ifaces/vlan<<VLAN_ID>>/options
echo "<<IP_ADDRESS>>/24" > /etc/net/ifaces/vlan<<VLAN_ID>>/ipv4address
echo "default via <<GATEWAY_IP>>" > /etc/net/ifaces/vlan<<VLAN_ID>>/ipv4route

```

### Switching (Bridge VLANs)

*Если нужно прокинуть VLAN 100 с bond0 на ens6:*

```bash
# 1. VLAN на входящем порту (bond0)
mkdir -p /etc/net/ifaces/bond0.100
echo "TYPE=vlan" > /etc/net/ifaces/bond0.100/options
echo "VID=100" >> /etc/net/ifaces/bond0.100/options
echo "HOST=bond0" >> /etc/net/ifaces/bond0.100/options

# 2. VLAN на исходящем порту (ens6)
mkdir -p /etc/net/ifaces/ens6.100
echo "TYPE=vlan" > /etc/net/ifaces/ens6.100/options
echo "VID=100" >> /etc/net/ifaces/ens6.100/options
echo "HOST=ens6" >> /etc/net/ifaces/ens6.100/options

# 3. Мост (Bridge)
mkdir -p /etc/net/ifaces/br100
echo "TYPE=bri" > /etc/net/ifaces/br100/options
# Перечисляем интерфейсы через пробел или новой строкой
echo "bond0.100 ens6.100" > /etc/net/ifaces/br100/ports

```

---

## 💾 4. Storage & Services (SRV2 - Target & RADIUS)

**Настройка сети SRV2:**
`/etc/net/ifaces/ens18/options` -> `TYPE=eth`
`/etc/net/ifaces/ens18/ipv4address` -> `<<IP_ADDRESS>>/24`
`/etc/net/ifaces/ens18/ipv4route` -> `default via <<GATEWAY>>`

### iSCSI Target

```bash
apt-get install targetcli
systemctl enable --now target

targetcli
# В консоли:
/backstores/block create name=disk1 dev=/dev/sdb
/iscsi create <<IQN_TARGET_NAME>>
/iscsi/<<IQN_TARGET_NAME>>/tpg1/acls create <<IQN_INITIATOR_NAME>>
/iscsi/<<IQN_TARGET_NAME>>/tpg1/luns create /backstores/block/disk1
exit

```

### RADIUS (FreeRADIUS)

```bash
apt-get install freeradius freeradius-utils

# /etc/raddb/clients.conf
client <<ROUTER_HOSTNAME>> {
    ipaddr = <<ROUTER_IP>>
    secret = <<RADIUS_SECRET>>
}

# /etc/raddb/users
<<USER>> Cleartext-Password := "<<PASS>>"
       Service-Type = Administrative-User

systemctl enable --now radiusd
# Тест:
radtest <<USER>> <<PASS>> localhost 0 testing123

```

---

## 🛠️ 5. Infrastructure (SRV1 - DNS, CA, Initiator)

### DNS (Bind)

```bash
apt-get install bind bind-utils

# /etc/bind/options.conf
options {
    listen-on { any; };
    allow-query { any; };
    recursion yes;
    forwarders { <<ISP_DNS_IP>>; };
    dnssec-validation no;
};

# /etc/bind/local.conf
zone "<<DOMAIN>>" IN {
    type master;
    file "/var/lib/bind/db.<<DOMAIN>>";
};
zone "<<REVERSE_ZONE>>.in-addr.arpa" IN {
    type master;
    file "/var/lib/bind/db.reverse";
};

# Файл зоны: /var/lib/bind/db.<<DOMAIN>>
$TTL 86400
@   IN  SOA     srv1.<<DOMAIN>>. root.<<DOMAIN>>. ( ... )
@   IN  NS      srv1.<<DOMAIN>>.
@   IN  A       <<SRV1_IP>>
srv1     IN A   <<SRV1_IP>>
monitoring IN CNAME srv1

```

### CA (OpenSSL)

```bash
mkdir -p /var/ca/{certs,crl,newcerts,private}
chmod 700 /var/ca/private
touch /var/ca/index.txt
echo 1000 > /var/ca/serial
cp /etc/ssl/openssl.cnf /var/ca/openssl.cnf

# Правка /var/ca/openssl.cnf:
# dir = /var/ca
# default_days = 1825

cd /var/ca
openssl genrsa -out private/ca.key 4096
openssl req -new -x509 -key private/ca.key -out ca.crt -config openssl.cnf
cp ca.crt /usr/share/ca-certificates/<<CA_NAME>>.crt
update-ca-trust

```

### iSCSI Initiator & NFS Server

```bash
apt-get install open-iscsi lvm2 nfs-utils

# 1. Connect
echo "InitiatorName=<<IQN_INITIATOR_NAME>>" > /etc/iscsi/initiatorname.iscsi
systemctl restart iscsid
iscsiadm -m discovery -t st -p <<SRV2_IP>>
iscsiadm -m node --login

# 2. LVM
pvcreate /dev/sdb
vgcreate VG /dev/sdb
lvcreate -l 100%FREE -n DATA VG
mkfs.xfs /dev/VG/DATA

# 3. Mount
mkdir -p /opt/data
# UUID="<<UUID>>" /opt/data xfs _netdev 0 0  >> /etc/fstab
mount -a

# 4. NFS Export
echo "/opt/data <<NETWORK_CIDR>>(rw,sync,no_root_squash)" >> /etc/exports
exportfs -ra
systemctl enable --now nfs-server

```

---

## 💻 6. Clients (Admin)

### NFS Mount & DB Client

```bash
apt-get install nfs-clients
mkdir -p /mnt/nfs
echo "<<SRV1_IP>>:/opt/data /mnt/nfs nfs _netdev 0 0" >> /etc/fstab
mount -a

```

**DBeaver Connection:**

* Host: `<<SRV2_IP>>`
* Database: `postgres` (or `zabbix`)
* User: `superadmin`
* Pass: `<<PASS>>`

---

## 🏢 7. Active Directory (Samba DC)

```bash
# 1. Provision
hostnamectl set-hostname <<DC_HOSTNAME>>
apt-get install samba-dc bind bind-utils
rm -f /etc/samba/smb.conf

samba-tool domain provision \
  --realm=<<REALM>> \
  --domain=<<DOMAIN_SHORT>> \
  --adminpass=<<PASS>> \
  --server-role=dc \
  --dns-backend=BIND9_DLZ

# 2. Config
# В named.conf:
# include "/var/lib/samba/bind-dns/named.conf";
# tkey-gssapi-keytab "/var/lib/samba/private/dns.keytab";

chgrp named /var/lib/samba/private/dns.keytab
chmod g+r /var/lib/samba/private/dns.keytab

systemctl disable smb nmb
systemctl unmask samba-ad-dc
systemctl enable --now samba-ad-dc
systemctl restart bind

# 3. Users & Groups
samba-tool ou create "OU=ofadmins"
samba-tool user create <<USER>> <<PASS>> --userou="OU=ofadmins"
samba-tool group addmembers <<GROUP>> <<USER>>

```

### Domain Client (cli1-a)

```bash
echo "nameserver <<DC_IP>>" > /etc/resolv.conf
system-auth write ad domain <<REALM>> computer <<HOSTNAME>> login administrator password <<PASS>>
systemctl enable --now winbind

```

---

## 📊 8. Zabbix (Monitoring)

### Database (SRV2)

```bash
# pg_hba.conf -> host all all <<NET>> scram-sha-256
su - postgres
psql -c "create user zabbix_user with password '<<PASS>>';"
psql -c "create database zabbix owner zabbix_user;"

```

### Server (SRV1)

```bash
apt-get install zabbix-server-pgsql zabbix-web-apache-pgsql zabbix-agent

# Import Schema (Run on SRV1, target SRV2 IP)
zcat .../create.sql.gz | psql -h <<SRV2_IP>> -U zabbix_user -d zabbix

# zabbix_server.conf
DBHost=<<SRV2_IP>>
DBUser=zabbix_user
DBPassword=<<PASS>>

# SSL (Apache)
# sites-available/ssl.conf: SSLCertificateFile /var/ca/certs/...
systemctl enable --now httpd2 zabbix-server

```

---

## 📞 9. Telephony (Asterisk)

**Web Interface:** `http://<<SIP_SERVER_IP>>`

1. **Extensions:** Create `1001`, `1002`, etc.
2. **Settings -> Asterisk SIP Settings:**
* **PJSIP:** Port `5160` (Disable/Change)
* **Chan_SIP:** Port `5060` (Enable/Bind)


3. **Clients:** Linphone -> `1001@<<SIP_SERVER_IP>>`
