enable
configure terminal

! 1. Базовая настройка
hostname rtr-cod
domain-name <<DOMAIN_NAME>>

! 2. Внешний интерфейс (ISP)
interface gigabitethernet 0/0
 description WAN_TO_ISP
 ip address <<WAN_IP>> <<WAN_MASK>>
 no shutdown
exit

! 3. Внутренний интерфейс (LAN)
interface gigabitethernet 0/1
 description LAN_TO_FW
 ip address <<LAN_IP>> <<LAN_MASK>>
 no shutdown
exit

! 4. BGP
router bgp <<MY_AS>>
 bgp router-id <<WAN_IP>>
 neighbor <<ISP_GW>> remote-as <<ISP_AS>>
 neighbor <<ISP_GW>> description ISP_UPLINK
 
 address-family ipv4
  neighbor <<ISP_GW>> activate
  ! Анонс СВОЕЙ сети (адрес сети, не IP интерфейса!)
  network <<WAN_NETWORK_ADDR>> mask <<WAN_MASK>>
 exit
exit
! Если маршрут 0.0.0.0/0 не приходит сам:
neighbor <<ISP_GW>> default-originate

! 5. GRE Tunnel
interface Tunnel1
 ip address <<TUNNEL_LOCAL_IP>> 255.255.255.252
 tunnel source <<WAN_IP>>
 tunnel destination <<REMOTE_ROUTER_WAN_IP>>
 tunnel mode gre ip
 ip ospf message-digest-key 1 md5 <<PASSWORD>>
exit

! 6. OSPF
router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface Tunnel1
 no passive-interface gigabitethernet 0/1
 ! Обратная маска (Wildcard)! /30 -> 0.0.0.3, /24 -> 0.0.0.255
 network <<TUNNEL_NETWORK>> 0.0.0.3 area 0
 network <<LAN_NETWORK>> <<WILDCARD_MASK>> area 0
 area 0 authentication message-digest
exit
write


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

hostnamectl set-hostname <<HOSTNAME>>

# [cite_start]1. Агрегация (Bonding) [cite: 115-117]
mkdir -p /etc/net/ifaces/bond0
echo "TYPE=bond" > /etc/net/ifaces/bond0/options
echo "bond-mode 1" >> /etc/net/ifaces/bond0/options
echo "bond-miimon 100" >> /etc/net/ifaces/bond0/options
# Указываем физические порты
echo "<<INTERFACE_1>>" > /etc/net/ifaces/bond0/ports
echo "<<INTERFACE_2>>" >> /etc/net/ifaces/bond0/ports
echo "BOOTPROTO=static" > /etc/net/ifaces/bond0/ipv4address

# Чистка физических портов
mkdir -p /etc/net/ifaces/<<INTERFACE_1>> /etc/net/ifaces/<<INTERFACE_2>>
echo "TYPE=eth" | tee /etc/net/ifaces/<<INTERFACE_1>>/options /etc/net/ifaces/<<INTERFACE_2>>/options

# 2. VLAN Управления (L3 интерфейс)
mkdir -p /etc/net/ifaces/vlan<<VLAN_ID>>
echo "TYPE=vlan" > /etc/net/ifaces/vlan<<VLAN_ID>>/options
echo "VID=<<VLAN_ID>>" >> /etc/net/ifaces/vlan<<VLAN_ID>>/options
echo "HOST=bond0" >> /etc/net/ifaces/vlan<<VLAN_ID>>/options

# IP адрес свитча
echo "<<SWITCH_IP>>/24" > /etc/net/ifaces/vlan<<VLAN_ID>>/ipv4address
echo "default via <<MGMT_GATEWAY>>" > /etc/net/ifaces/vlan<<VLAN_ID>>/ipv4route

# 3. Проброс VLAN (L2 Bridge)
# Пример для VLAN Серверов
mkdir -p /etc/net/ifaces/bond0.<<SRV_VLAN_ID>>
echo "TYPE=vlan" > /etc/net/ifaces/bond0.<<SRV_VLAN_ID>>/options
echo "VID=<<SRV_VLAN_ID>>" >> /etc/net/ifaces/bond0.<<SRV_VLAN_ID>>/options
echo "HOST=bond0" >> /etc/net/ifaces/bond0.<<SRV_VLAN_ID>>/options

# Интерфейс, куда смотрят сервера
mkdir -p /etc/net/ifaces/<<SRV_IFACE>>.<<SRV_VLAN_ID>>
echo "TYPE=vlan" > /etc/net/ifaces/<<SRV_IFACE>>.<<SRV_VLAN_ID>>/options
echo "VID=<<SRV_VLAN_ID>>" >> /etc/net/ifaces/<<SRV_IFACE>>.<<SRV_VLAN_ID>>/options
echo "HOST=<<SRV_IFACE>>" >> /etc/net/ifaces/<<SRV_IFACE>>.<<SRV_VLAN_ID>>/options

# Мост
mkdir -p /etc/net/ifaces/br<<SRV_VLAN_ID>>
echo "TYPE=bri" > /etc/net/ifaces/br<<SRV_VLAN_ID>>/options
echo "bond0.<<SRV_VLAN_ID>>" > /etc/net/ifaces/br<<SRV_VLAN_ID>>/ports
echo "<<SRV_IFACE>>.<<SRV_VLAN_ID>>" >> /etc/net/ifaces/br<<SRV_VLAN_ID>>/ports

systemctl restart network

# 1. Сеть
mkdir -p /etc/net/ifaces/<<IFACE>>
echo "TYPE=eth" > /etc/net/ifaces/<<IFACE>>/options
echo "<<SRV2_IP>>/24" > /etc/net/ifaces/<<IFACE>>/ipv4address
echo "default via <<GATEWAY>>" > /etc/net/ifaces/<<IFACE>>/ipv4route
systemctl restart network

# [cite_start]2. Targetcli [cite: 173-177]
apt-get install targetcli
targetcli

/backstores/block create name=disk1 dev=/dev/sdb
/iscsi create <<IQN_TARGET_NAME>>
# ACL (Имя, которым представится srv1)
/iscsi/<<IQN_TARGET_NAME>>/tpg1/acls create <<IQN_INITIATOR_NAME>>
/iscsi/<<IQN_TARGET_NAME>>/tpg1/luns create /backstores/block/disk1
exit



# 1. Подключение
apt-get install open-iscsi lvm2 nfs-utils
echo "InitiatorName=<<IQN_INITIATOR_NAME>>" > /etc/iscsi/initiatorname.iscsi
systemctl restart iscsid

# Подключение к srv2
iscsiadm -m discovery -t st -p <<SRV2_IP>>
iscsiadm -m node --login

# [cite_start]2. LVM [cite: 183-189]
pvcreate /dev/sdb
vgcreate VG /dev/sdb
lvcreate -l 100%FREE -n DATA VG
mkfs.xfs /dev/VG/DATA

mkdir -p /opt/data
# UUID="<<UUID_OF_DISK>>" /opt/data xfs _netdev 0 0 >> /etc/fstab
mount -a

# [cite_start]3. NFS Export [cite: 190-192]
echo "/opt/data <<MGMT_NETWORK>>/24(rw,sync,no_root_squash)" >> /etc/exports
exportfs -ra
systemctl enable --now nfs-server


# 1. RADIUS (Clients)
# /etc/raddb/clients.conf
client <<ROUTER_HOSTNAME>> {
    ipaddr = <<ROUTER_IP>>
    secret = <<RADIUS_SECRET>>
}

# 2. RADIUS (Users)
# /etc/raddb/users
<<USER>> Cleartext-Password := "<<PASSWORD>>"
       Service-Type = Administrative-User

# [cite_start]3. DNS (Named) [cite: 144-154]
# /etc/bind/options.conf -> forwarders { <<NTP_DNS_SERVER_IP>>; };
# /etc/bind/local.conf -> zone "<<DOMAIN>>" { type master; file "..."; };

# Файл зоны
@   IN  SOA     srv1. root. ( ... )
@   IN  NS      srv1.
@   IN  A       <<SRV1_IP>>
srv1     IN A   <<SRV1_IP>>
srv2     IN A   <<SRV2_IP>>
router   IN A   <<ROUTER_IP>>
monitoring IN CNAME srv1

# [cite_start]4. CA (OpenSSL) [cite: 156-161]
# /var/ca/openssl.cnf -> dir = /var/ca
openssl genrsa -out private/ca.key 4096
openssl req -new -x509 -key private/ca.key -out ca.crt -days 1825 -config openssl.cnf
cp ca.crt /usr/share/ca-certificates/<<NAME>>.crt
update-ca-trust


# [cite_start]Provisioning [cite: 195-199]
samba-tool domain provision \
  --realm=<<REALM_UPPERCASE>> \
  --domain=<<DOMAIN_SHORT>> \
  --adminpass=<<PASSWORD>> \
  --server-role=dc \
  --dns-backend=BIND9_DLZ

# В named.conf
include "/var/lib/samba/bind-dns/named.conf";

# [cite_start]Создание объектов [cite: 201-208]
samba-tool ou create "OU=<<OU_NAME>>"
samba-tool user create <<USER_LOGIN>> <<PASSWORD>> --userou="OU=<<OU_NAME>>"
samba-tool group addmembers <<GROUP_NAME>> <<USER_LOGIN>>


# pg_hba.conf -> host all all <<NETWORK>> scram-sha-256
create user <<DB_USER>> with password '<<DB_PASS>>';
create database zabbix owner <<DB_USER>>;


# Импорт схемы (запуск на srv1, заливка на srv2)
zcat .../create.sql.gz | psql -h <<SRV2_IP>> -U <<DB_USER>> -d zabbix

# zabbix_server.conf
DBHost=<<SRV2_IP>>
DBUser=<<DB_USER>>
DBPassword=<<DB_PASS>>
