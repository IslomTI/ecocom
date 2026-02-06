Вот твой готовый файл для GitHub.

**Что я сделал:**

1. **Исправил мелкие ошибки:**
* В BGP команда `network` объявляет **адрес сети** (например, `x.x.x.0`), а не адрес шлюза (`.1`). Поправил на `<<network-addr>>`.
* Разделил установку пакетов (targetcli и freeradius), чтобы не было каши.
* Уточнил синтаксис `ipv4address` (там нужен `/mask` в формате CIDR, например `/24`).


2. **Оформление:**
* Поскольку GitHub Markdown **не поддерживает** цветной текст (синий/красный) напрямую в коде, я использовал **синтаксическую подсветку**.
* `<<...>>` будут выделяться как переменные.
* `->` и комментарии будут серыми или зелеными (в зависимости от темы GitHub).
* Сохранил твою структуру: "Команда/Файл -> Содержимое".



Скопируй код ниже в свой `README.md`.

---

```markdown
# 🚀 Network & System Admin Cheat Sheet

> **Легенда:**
> * `<<...>>` — (Синий) Переменные, которые нужно подставить из задания.
> * `->` — (Красный) Действие или результат.
> * `''' ... '''` — Общие команды / Конфигурация файлов.

---

## 1. 🟢 EcoRouter (Network)

```cisco
enable
configure terminal

! Имя устройства
hostname <<hostname>>
domain-name <<domain-name>>

! Настройка интерфейсов
interface gigabitethernet <<int-(0/0)|(0/1.vlan)>>
 description <<net-name>> | encapsulation dot1Q <<vlan>>
 ip address <<ip-address>> <<mask>>
 no shutdown
exit

! Настройка BGP с провайдером
router bgp <<big-AS>>
 bgp router-id <<ip-address>>
 neighbor <<gateway>> remote-as <<AS>>
 neighbor <<gateway>> description <<AS-name>>
 address-family ipv4
  neighbor <<gateway>> activate
  ! Объявляем СВОЮ сеть (не шлюз!), маска должна совпадать
  network <<network-address>> mask <<mask>>
 exit
exit

! Дефолт, если не прилетел
neighbor <<gateway>> default-originate

! Настройка GRE туннеля
interface Tunnel1
 ip address <<ip-address-pk>> <<mask>>
 tunnel source <<ip-address-local>>
 tunnel destination <<ip-address-remote>>
 tunnel mode gre ip
 ip ospf message-digest-key 1 md5 <<password>>
exit

! Настройка OSPF
router ospf 1
 router-id <<ip-address-(1.1.1.1)>>
 passive-interface default
 no passive-interface Tunnel1
 no passive-interface gigabitethernet <<int-(0/1)|(0/1.300)>>
 ! Сети объявляем с Wildcard маской (0.0.0.3 или 0.0.0.255)
 network 10.10.10.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.3 area 0
 area 0 authentication message-digest
exit
write

```

---

## 2. 🟢 EcoRouter (RADIUS Client)

```cisco
enable
conf t

aaa new-model

! Указываем IP сервера RADIUS (srv1)
radius-server host 192.168.100.10 key radius_secret

aaa authentication login default group radius local

username admin privilege 15 secret P@ssw0rdLocal

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

## 3. 🐧 Alt Linux (Network Configs)

```bash
hostnamectl set-hostname <<host-name>>

# --- Bond0 (active-backup) ---
nano /etc/net/ifaces/bond0/options
'''
TYPE=bond
bond-mode 1
bond-miimon 100
'''
nano /etc/net/ifaces/bond0/ports
'''
ens4
ens5
'''
# Статика на бонде
/etc/net/ifaces/bond0/ipv4address -> BOOTPROTO=static

# Чистка физических портов
rm -rf /etc/net/ifaces/ens4
rm -rf /etc/net/ifaces/ens5

# --- VLAN 300 (MGMT / L3) ---
nano /etc/net/ifaces/vlan300/options
'''
TYPE=vlan
VID=300
HOST=bond0
'''
/etc/net/ifaces/vlan300/ipv4address -> <<ip-address-switch>>/24
/etc/net/ifaces/vlan300/ipv4route -> default via <<gateway-ip>>

# --- VLAN 100 (Servers / L2 Bridge) ---
# 1. VLAN на бонде
nano /etc/net/ifaces/bond0.100/options
'''
TYPE=vlan
VID=100
HOST=bond0
'''

# 2. VLAN на порту серверов (ens6)
nano /etc/net/ifaces/ens6.100/options
'''
TYPE=vlan
VID=100
HOST=ens6
'''

# 3. Мост br100
nano /etc/net/ifaces/br100/options
'''
TYPE=bri
PORTS="bond0.100 ens6.100"
'''

# Применить
systemctl restart network
ip a

```

---

## 4. 💾 iSCSI (Storage)

### 📤 SRV2 (Target - Сервер)

```bash
# Сеть
/etc/net/ifaces/ens18/options -> TYPE=eth
/etc/net/ifaces/ens18/ipv4address -> <<ip-address>>/24
/etc/net/ifaces/ens18/ipv4route -> default via <<gateway>>
systemctl restart network

# Установка
apt-get install targetcli
systemctl enable --now target

# Конфигурация
targetcli
'''
/backstores/block create name=disk1 dev=/dev/sdb
# Создаем цель
/iscsi create iqn.2026-01.region.ssa2026.cod:data.target
# ACL (Разрешаем клиенту srv1)
/iscsi/iqn.2026-01.region.ssa2026.cod:data.target/tpg1/acls create iqn.2026-01.region.ssa2026.cod:iscsi
# Привязываем диск
/iscsi/iqn.2026-01.region.ssa2026.cod:data.target/tpg1/luns create /backstores/block/disk1
exit
'''

```

### 📥 SRV1 (Initiator - Клиент)

```bash
apt-get install open-iscsi lvm2 nfs-utils

# Имя инициатора (должно совпадать с ACL на сервере!)
echo "InitiatorName=iqn.2026-01.region.ssa2026.cod:iscsi" > /etc/iscsi/initiatorname.iscsi
systemctl restart iscsid

# Подключение
iscsiadm -m discovery -t st -p 192.168.100.11
iscsiadm -m node --login

# LVM
pvcreate /dev/sdb
vgcreate VG /dev/sdb
lvcreate -l 100%FREE -n DATA VG
mkfs.xfs /dev/VG/DATA

# Монтирование
mkdir -p /opt/data
# Узнаем UUID: blkid /dev/VG/DATA
# fstab: UUID="xxxx" /opt/data xfs _netdev 0 0
mount -a

# NFS Export
echo "/opt/data 10.10.30.0/24(rw,sync,no_root_squash)" >> /etc/exports
exportfs -ra
systemctl enable --now nfs-server

```

---

## 5. 🛠️ Services (Infrastructure)

### 🛡️ RADIUS (srv1)

```bash
apt-get install freeradius freeradius-utils

nano /etc/raddb/clients.conf
'''
client rtr-cod {
    ipaddr = 10.10.30.1
    secret = radius_secret
}
client sw1-cod {
    ipaddr = 10.10.30.11
    secret = radius_secret
}
'''

nano /etc/raddb/users
'''
netuser Cleartext-Password := "P@ssw0rd"
       Service-Type = Administrative-User
'''

systemctl enable --now radiusd
# Проверка
radtest netuser P@ssw0rd localhost 0 testing123

```

### 🌐 DNS (srv1)

```bash
apt-get install bind bind-utils

nano /etc/bind/options.conf
'''
options {
    listen-on { any; };
    allow-query { any; };
    recursion yes;
    forwarders { 100.100.100.100; };
    dnssec-validation no;
};
'''

nano /etc/bind/local.conf
'''
zone "cod.ssa2026.region" IN {
    type master;
    file "/var/lib/bind/cod.ssa2026.region.db";
};
zone "100.168.192.in-addr.arpa" IN {
    type master;
    file "/var/lib/bind/100.168.192.db";
};
'''

nano /var/lib/bind/cod.ssa2026.region.db
'''
$TTL 86400
@   IN  SOA     srv1-cod.cod.ssa2026.region. root.cod.ssa2026.region. (
        2026012801 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

@       IN  NS      srv1-cod.cod.ssa2026.region.
@       IN  A       192.168.100.10

; Записи
srv1-cod IN  A       192.168.100.10
srv2-cod IN  A       192.168.100.11
fw-cod   IN  A       192.168.100.1
rtr-cod  IN  A       10.10.30.1
sw1-cod  IN  A       10.10.30.11
sw2-cod  IN  A       10.10.30.12
sip-cod  IN  A       192.168.100.20
admin-cod IN A       10.10.30.100
monitoring IN CNAME  srv1-cod
'''
systemctl enable --now bind

```

### 🔐 CA (OpenSSL)

```bash
mkdir -p /var/ca/{certs,crl,newcerts,private}
chmod 700 /var/ca/private
touch /var/ca/index.txt
echo 1000 > /var/ca/serial
cp /etc/ssl/openssl.cnf /var/ca/openssl.cnf

nano /var/ca/openssl.cnf
# Правки:
# dir = /var/ca
# default_days = 1825
# organizationName_default = IRPO
# commonName_default = ssa2026

cd /var/ca
openssl genrsa -out private/ca.key 4096
openssl req -new -x509 -key private/ca.key -out ca.crt -days 1825 -config openssl.cnf
cp ca.crt /usr/share/ca-certificates/ssa2026.crt
update-ca-trust

```

---

## 6. 🏢 Active Directory (DC-A)

```bash
hostnamectl set-hostname dc-a.office.ssa2026.region
# /etc/hosts -> 192.168.100.10 dc-a...

apt-get install samba-dc bind bind-utils
rm -f /etc/samba/smb.conf

# Provision
samba-tool domain provision \
  --realm=OFFICE.SSA2026.REGION \
  --domain=OFFICE \
  --adminpass=P@ssw0rd \
  --server-role=dc \
  --dns-backend=BIND9_DLZ

# Named Config
nano /etc/bind/named.conf
'''
tkey-gssapi-keytab "/var/lib/samba/private/dns.keytab";
include "/var/lib/samba/bind-dns/named.conf";
'''

# Права
chgrp named /var/lib/samba/private/dns.keytab
chmod g+r /var/lib/samba/private/dns.keytab

systemctl disable smb nmb
systemctl enable --now samba-ad-dc
systemctl restart bind

# Пользователи и Группы
samba-tool ou create "OU=ofadmins"
samba-tool group add ofadmins --groupou="OU=ofadmins"
samba-tool user create ofadmin1 P@ssw0rd --userou="OU=ofadmins"
samba-tool group addmembers ofadmins ofadmin1

```

---

## 7. 📊 Zabbix & Clients

### PostgreSQL (srv2)

```bash
# Графика (DBeaver):
# Host: 192.168.100.11, User: superadmin, Pass: P@ssw0rdSQL

```

### Zabbix Server (srv1)

```bash
apt-get install zabbix-server-pgsql zabbix-web-apache-pgsql zabbix-agent

# Импорт БД (ввод пароля P@ssw0rdZabbix)
zcat /usr/share/doc/zabbix-server-pgsql-*/create.sql.gz | psql -h 192.168.100.11 -U zabbix_user -d zabbix

# Конфиг сервера
nano /etc/zabbix/zabbix_server.conf
'''
DBHost=192.168.100.11
DBName=zabbix
DBUser=zabbix_user
DBPassword=P@ssw0rdZabbix
'''

# SSL Apache
nano /etc/httpd2/conf/sites-available/ssl.conf
'''
SSLEngine on
SSLCertificateFile /var/ca/certs/srv1-cod.crt
SSLCertificateKeyFile /var/ca/private/srv1-cod.key
'''

systemctl enable --now httpd2 zabbix-server zabbix-agent

```

### Zabbix Agent (Клиенты)

```bash
nano /etc/zabbix/zabbix_agentd.conf
'''
Server=192.168.100.10
ServerActive=192.168.100.10
Hostname=<<REAL_HOSTNAME>>
'''

```

### EcoRouter SNMP

```cisco
snmp-server community public ro

```

```

```
