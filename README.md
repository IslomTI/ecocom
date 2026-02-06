enable
configure terminal

! 1. База
hostname <<ИМЯ_РОУТЕРА>>
domain-name <<ДОМЕН>>

! 2. Внешний интерфейс (Смотришь в лист задания)
interface gigabitethernet 0/0
 description WAN
 ip address <<МОЙ_ВНЕШНИЙ_IP>> <<МАСКА_ОБЫЧНАЯ>>
 no shutdown
exit

! 3. Внутренний интерфейс (Придумываешь сам или по схеме)
interface gigabitethernet 0/1
 description LAN
 ip address <<МОЙ_ВНУТРЕННИЙ_IP>> <<МАСКА_ОБЫЧНАЯ>>
 no shutdown
exit

! 4. BGP (Интернет)
! Логика: Я (моя AS) дружу с Соседом (его AS)
router bgp <<МОЯ_AS>>
 bgp router-id <<МОЙ_ВНЕШНИЙ_IP>>
 
 ! Настройка соседа
 neighbor <<IP_ШЛЮЗА_ISP>> remote-as <<AS_ПРОВАЙДЕРА>>
 neighbor <<IP_ШЛЮЗА_ISP>> description ISP
 
 address-family ipv4
  neighbor <<IP_ШЛЮЗА_ISP>> activate
  ! Важно: Анонсируем СВОЮ сеть (не шлюз, а адрес сети, обычно заканчивается на 0 или 8, 16...)
  network <<АДРЕС_МОЕЙ_ВНЕШНЕЙ_СЕТИ>> mask <<МАСКА_ОБЫЧНАЯ>>
 exit
exit
! Если инета нет, проверь пинг до шлюза. Команду default-originate тут писать НЕ НАДО, её пишет провайдер.

! 5. Туннель (Связь офисов)
interface Tunnel1
 ! IP внутри туннеля (придумал, например 10.10.10.1)
 ip address <<IP_ТУННЕЛЯ>> 255.255.255.252
 ! От кого летит (мой внешний IP)
 tunnel source <<МОЙ_ВНЕШНИЙ_IP>>
 ! Куда летит (внешний IP второго роутера)
 tunnel destination <<ВНЕШНИЙ_IP_ДРУГА>>
 tunnel mode gre ip
 ! Пароль для OSPF
 ip ospf message-digest-key 1 md5 <<ПАРОЛЬ>>
exit

! 6. OSPF (Маршруты внутри)
router ospf 1
 router-id 1.1.1.1
 passive-interface default
 ! Открываем рот только там, где свои (Туннель и ЛВС)
 no passive-interface Tunnel1
 no passive-interface gigabitethernet 0/1
 
 ! Объявляем сети. ВНИМАНИЕ: Тут обратная маска (Wildcard)!
 ! Если маска 255.255.255.252 (/30) -> Wildcard 0.0.0.3
 ! Если маска 255.255.255.0 (/24)   -> Wildcard 0.0.0.255
 network <<СЕТЬ_ТУННЕЛЯ>> 0.0.0.3 area 0
 network <<СЕТЬ_ЛВС>> 0.0.0.3 area 0
 
 area 0 authentication message-digest
exit
