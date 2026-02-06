# EсoRouters
<1>
```
enable
configure terminal
```
Имя устройства
```
hostname <<hostname>>
domain-name <<domain-name>>
interface gigabitethernet <<int-(0/0)|(0/1.vlan)>>
 description <<net-name>> | encapsulation dot1Q <<vlan>>
 ip address <<ip-adress <<mask>>
 no shutdown
exit
```
Настройка BGP с провайдером
```
router bgp <<big-AS>>
 bgp router-id <<ip-adress>>
 neighbor <<gateway>> remote-as <<AS>>
 neighbor <<gateway>> description <<AS-name>>
 address-family ipv4
  neighbor <<gateway>> activate
  network <<gateway-1>> mask <<mask>>
 exit
exit

Дефолт, если не прилетел
neighbor 178.207.179.1 default-originate
```
Настройка GRE туннеля до офиса
```
interface Tunnel1
 ip address <<ip-address-pk>> <<mask>>
 tunnel source <<ip-adress>>
 tunnel destination <<ip-address-2>>
 tunnel mode gre ip
```
Настройка OSPF
```
ip ospf message-digest-key 1 md5 <<password>>
router ospf 1
 router-id <<ip-adress-(1.1.1.1)>>
 passive-interface default
 no passive-interface Tunnel1
 no passive-interface gigabitethernet <<int-(0/1)|(0/1.300)>>
 | area 0 authentication message-digest
 network 10.10.10.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.3 area 0
 area 0 authentication message-digest
exit
write
```
<1>
# Ecorouter (RADIUS Client)
```
enable
conf t
aaa new-model
radius-server host <<ip-adress-srv1>> key radius_secret
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
# Alt Linuxs
<2>
```
hostnamectl set-hostname <<host-name>>
```
Создаем Bond0 (active-backup)
```
nano /etc/net/ifaces/bond0/options
TYPE=bond
bond-mode 1
bond-miimon 100
nano /etc/net/ifaces/bond0/ports
ens4
ens5
/etc/net/ifaces/bond0/ipv4address -> BOOTPROTO=static
rm -rf /etc/net/ifaces/ens4
rm -rf /etc/net/ifaces/ens5
```
Vlan
```
nano /etc/net/ifaces/vlan300/options
TYPE=vlan
VID=<<vlan>>
HOST=bond0
/etc/net/ifaces/vlan<<vlan>>/ipv4address -> <<ip-adress-rout/mask32>>
/etc/net/ifaces/vlan<<vlan>>/ipv4route -> default via <<gateway-rout>>
```
<2>
VLAN на порту серверов (ens6)
```
nano /etc/net/ifaces/ens6.<<vlan>>/options
TYPE=vlan
VID=<<vlan>>
HOST=ens6
```
Мост br100
```
nano /etc/net/ifaces/br100/options
TYPE=bri
bond0.100
ens6.100
```
<3>
```
systemctl restart network
ip a
```
<3>
# srv1-2-cod (Alt Linux) - iSCSI Target
<4>
```
/etc/net/ifaces/ens18/options -> TYPE=eth
/etc/net/ifaces/ens18/ipv4address -> <<ip-adress/mask32>>
/etc/net/ifaces/ens18/ipv4route -> default via <<gateway>>
systemctl restart network
```
<4>
```
apt-get install targetcli | freeradius freeradius-utils
systemctl enable --now target
```
targetcli
```
/backstores/block create name=disk1 dev=/dev/sdb
# Создаем цель (Target)
/iscsi create iqn.2026-01.region.ssa2026.cod:data.target
# Разрешаем доступ клиенту (ACL)
/iscsi/iqn.2026-01.region.ssa2026.cod:data.target/tpg1/acls create iqn.2026-01.region.ssa2026.cod:iscsi
# Привязываем диск
/iscsi/iqn.2026-01.region.ssa2026.cod:data.target/tpg1/luns create /backstores/block/disk1
exit
```
# RADIUS
```
nano /etc/raddb/clients.conf
client rtr-cod {
    ipaddr = <<ip-adress>>
    secret = radius_secret
}
client sw1-cod {
    ipaddr = <<ip-adress>>
    secret = radius_secret
}
```
```
nano /etc/raddb/users
netuser Cleartext-Password := "P@ssw0rd"
       Service-Type = Administrative-User

systemctl enable --now radiusd
radtest netuser P@ssw0rd localhost 0 testing123
```
# DNS
```
apt-get install bind bind-utils
```
```
nano /etc/bind/options.conf
options {
    listen-on { any; };
    allow-query { any; };
    recursion yes;
    forwarders { <<DNS>>; };
    dnssec-validation no;
};
```
```
nano /etc/bind/local.conf
zone "cod.ssa2026.region" IN {
    type master;
    file "/var/lib/bind/cod.ssa2026.region.db";
};
zone "100.168.192.in-addr.arpa" IN {
    type master;
    file "/var/lib/bind/100.168.192.db";
};
```
```
nano /var/lib/bind/cod.ssa2026.region.db
$TTL 86400
@   IN  SOA     srv1-cod.cod.ssa2026.region. root.cod.ssa2026.region. (
        2026012801 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

@       IN  NS      srv1-cod.cod.ssa2026.region.
@       IN  A       192.168.100.10

; Записи для устройств
srv1-cod IN  A       192.168.100.10
srv2-cod IN  A       192.168.100.11
fw-cod   IN  A       192.168.100.1
rtr-cod  IN  A       10.10.30.1
sw1-cod  IN  A       10.10.30.11
sw2-cod  IN  A       10.10.30.12
sip-cod  IN  A       192.168.100.20
admin-cod IN A       10.10.30.100
monitoring IN CNAME  srv1-cod
```

Проверка: `systemctl enable --now bind`

```
mkdir -p /var/ca/{certs,crl,newcerts,private}
chmod 700 /var/ca/private
touch /var/ca/index.txt
echo 1000 > /var/ca/serial
cp /etc/ssl/openssl.cnf /var/ca/openssl.cnf
```
```
nano /var/ca/openssl.cnf
* `dir = /var/ca`
* `default_days = 1825`
* `organizationName_default = IRPO`
* `commonName_default = ssa2026`
```
```
cd /var/ca
openssl genrsa -out private/ca.key 4096
openssl req -new -x509 -key private/ca.key -out ca.crt -days 1825 -config openssl.cnf
cp ca.crt /usr/share/ca-certificates/ssa2026.crt
update-ca-trust
```
```
apt-get install open-iscsi lvm2 nfs-utils
echo "InitiatorName=iqn.2026-01.region.ssa2026.cod:iscsi" > /etc/iscsi/initiatorname.iscsi
systemctl restart iscsid
```
```
iscsiadm -m discovery -t st -p 192.168.100.11
iscsiadm -m node --login
```
```
pvcreate /dev/sdb
vgcreate VG /dev/sdb
lvcreate -l 100%FREE -n DATA VG
```
```
mkdir -p /opt/data
# Узнаем UUID: blkid /dev/VG/DATA
# Добавляем в fstab: UUID="xxxx" /opt/data xfs _netdev 0 0
mount -a
```
# NFS
```
echo "/opt/data 10.10.30.0/24(rw,sync,no_root_squash)" >> /etc/exports
exportfs -ra
systemctl enable --now nfs-server
```
```
apt-get update
apt-get install nfs-clients
mkdir -p /mnt/nfs
```
Настройка авто-монтирования (`/etc/fstab`)
```
# Добавляем строку: IP_SRV1:/path /local_path nfs _netdev 0 0
echo "10.10.30.11:/opt/data /mnt/nfs nfs _netdev 0 0" >> /etc/fstab
mount -a
```
2 Подключение к БД (DBeaver) 
*Выполняется в графическом интерфейсе*
* **Host**: 192.168.100.11 (srv2-cod)
* **Database**: postgres
* **User**: superadmin
* **Password**: P@ssw0rdSQL
*

# dc-a (Samba AD DC)
```
hostnamectl set-hostname dc-a.office.ssa2026.region
```
Проверь /etc/hosts: 192.168.100.10 dc-a.office.ssa2026.region dc-a
```
apt-get install samba-dc bind bind-utils
rm -f /etc/samba/smb.conf
```
```
samba-tool domain provision \
  --realm=OFFICE.SSA2026.REGION \
  --domain=OFFICE \
  --adminpass=P@ssw0rd \
  --server-role=dc \
  --dns-backend=BIND9_DLZ
```
В файл `/etc/bind/named.conf` (или options) добавь:

```
tkey-gssapi-keytab "/var/lib/samba/private/dns.keytab";
# В конец файла:
include "/var/lib/samba/bind-dns/named.conf";
```
```
chgrp named /var/lib/samba/private/dns.keytab
chmod g+r /var/lib/samba/private/dns.keytab
```
```
systemctl stop smb nmb
systemctl disable smb nmb
systemctl unmask samba-ad-dc
systemctl enable --now samba-ad-dc
systemctl restart bind
```
Подразделения
```
samba-tool ou create "OU=ofadmins"
samba-tool ou create "OU=ofusers"
```
Группы
```
samba-tool group add ofadmins --groupou="OU=ofadmins"
samba-tool group add ofusers --groupou="OU=ofusers"
```
Пользователи
```
samba-tool user create ofadmin1 P@ssw0rd --userou="OU=ofadmins"
samba-tool user create ofuser1 P@ssw0rd --userou="OU=ofusers"
samba-tool user create user1 P@ssw0rd
```
Добавление в группы
```
samba-tool group addmembers ofadmins ofadmin1
samba-tool group addmembers ofusers ofuser1
```
# cli1-a (Client)

Указываем DNS на контроллер домена
```
echo "nameserver 192.168.100.10" > /etc/resolv.conf
system-auth write ad domain office.ssa2026.region computer cli1-a login administrator password P@ssw0rd
systemctl enable --now winbind
```
2 Групповые политики (ADMC) 
*Выполняется в графике под ofadmin1*
* Запустить `admc`.
* Подключиться к домену.
* GPO -> User Configuration -> Desktop -> Wallpaper (запретить смену, установить картинку).
* GPO -> User Configuration -> Network -> Prohibit changes.


**Настройка на srv1-cod** 
```
apt-get install zabbix-server-pgsql zabbix-web-apache-pgsql zabbix-agent
```
# Вводим пароль P@ssw0rdZabbix
zcat /usr/share/doc/zabbix-server-pgsql-*/create.sql.gz | psql -h 192.168.100.11 -U zabbix_user -d zabbix
```
nano /etc/zabbix/zabbix_server.conf
DBHost=192.168.100.11
DBName=zabbix
DBUser=zabbix_user
DBPassword=P@ssw0rdZabbix
```
```
nano /etc/httpd2/conf/sites-available/ssl.conf
SSLEngine on
SSLCertificateFile /var/ca/certs/srv1-cod.crt
SSLCertificateKeyFile /var/ca/private/srv1-cod.key
systemctl enable --now httpd2 zabbix-server zabbix-agent
```
nano /etc/zabbix/zabbix_agentd.conf
```
Server=192.168.100.10
ServerActive=192.168.100.10
Hostname=<ИМЯ_ЭТОЙ_МАШИНЫ>
```

**Настройка SNMP (EcoRouter)** 

snmp-server community public ro

# sip-cod (IP Telephony)

*Настройка через Web-интерфейс (http://IP-SIP-COD)*

1. 
**Создать Extensions (Chan_SIP)** :


* 1001 (admin-cod)
* 1002 (cli-cod)
* 2001 (cli1-a)
* 2002 (cli2-a)
* Secret: `P@ssw0rd`


2. 
**Настройка портов**:


* Settings -> Asterisk SIP Settings.
* **Chan SIP**: Bind Port `5060`.
* **PJSIP**: Bind Port `5160` (или отключить).


3. 
**Софтфоны (на клиентах)**:


* Установить `linphone`.
* Вход: `1001@IP_SIP_COD`, пароль `P@ssw0rd`.
