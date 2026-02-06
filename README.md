# 🏆 Cheat Sheet: Network & System Administration (Module B)

Шпаргалка для подготовки к региональному чемпионату "Профессионалы".
**Стек:** Alt Linux 10/11, EcoRouter, PostgreSQL, Zabbix, Samba AD.

---

## 📑 Содержание
1. [Маршрутизация (EcoRouter)](#1-маршрутизация-ecorouter)
2. [Коммутация (Alt Linux)](#2-коммутация-alt-linux)
3. [Хранение данных (iSCSI & NFS)](#3-хранение-данных-iscsi--nfs)
4. [Инфраструктура (DNS, CA, RADIUS)](#4-инфраструктура-dns-ca-radius)
5. [Офис и Домен (Samba AD)](#5-офис-и-домен-samba-ad)
6. [Мониторинг (Zabbix)](#6-мониторинг-zabbix)
7. [Телефония (Asterisk)](#7-телефония-asterisk)

---

## 1. Маршрутизация (EcoRouter)

### 📍 rtr-cod (Центральный роутер)
*IP: 178.207.179.4/29, AS 64500*

```bash
enable
configure terminal

! 1. Базовая настройка
hostname rtr-cod
domain-name cod.ssa2026.region

! 2. Внешний интерфейс (ISP)
interface gigabitethernet 0/0
 description WAN_TO_ISP
 ip address 178.207.179.4 255.255.255.248
 no shutdown
exit

! 3. Внутренний интерфейс (к FW)
interface gigabitethernet 0/1
 description LAN_TO_FW
 ! Адрес транзитной сети до FW (придуманный)
 ip address 10.0.1.1 255.255.255.252
 no shutdown
exit

! 4. BGP с провайдером
router bgp 64500
 bgp router-id 178.207.179.4
 neighbor 178.207.179.1 remote-as 31133
 neighbor 178.207.179.1 description ISP_UPLINK
 
 address-family ipv4
  neighbor 178.207.179.1 activate
  ! Анонсируем СВОЮ внешнюю сеть (важно совпадение маски!)
  network 178.207.179.0 mask 255.255.255.248
 exit
exit
! Если маршрут не прилетает сам:
neighbor 178.207.179.1 default-originate

! 5. GRE Tunnel до офиса
interface Tunnel1
 ip address 10.10.10.1 255.255.255.252
 tunnel source 178.207.179.4
 tunnel destination 178.207.179.28
 tunnel mode gre ip
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit

! 6. OSPF (Внутренняя маршрутизация)
router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface Tunnel1
 no passive-interface gigabitethernet 0/1
 ! Wildcard маски: /30 -> 0.0.0.3
 network 10.10.10.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.3 area 0
 area 0 authentication message-digest
exit
write
