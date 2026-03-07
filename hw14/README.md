# Домашнее задание №7 «VXLAN. Multihoming»

## Цель

Настроить отказоустойчивое подключение клиентов с использованием EVPN Multihoming.

* [1. Подключение клиентов.](#1-подключение-клиентов)
* [2. Настройка агрегации в сторону клиентов.](#2-настройка-агрегации-в-сторону-клиентов)
* [3. Multihoming ESI LAG.](#3-multihoming-esi-lag)
* [4. Multihoming MC-LAG.](#4-multihoming-mc-lag)

## Топология

Топология лабораторного стенда собрана в среде EVE-NG. Для работы мультихоминга
внесены изменения: добавлен ещё один лиф L4 (соединён с L3 в MC-LAG пару) и
дополнительные линки между клиентами и лифами.

![Топология с IP адресами](topology.drawio.png)

## 1. Подключение клиентов

В качестве основы возьмём предыдущую лабораторную работу по L3 EVPN (eBGP over eBGP),
таким образом часть настроек уже выполнена и останется лишь внести изменения.

### Предварительная донастройка спайнов и лифов

Добавим некоторые дополнительные настройки: включим на bgp пирах аутентификацию
и BFD. Также увеличим mtu на линках между лифами и спайнами (mtu 9214).

Пример команд для первого спайна:

```text
enable
conf t
router bgp 65500
neighbor EVPN bfd
neighbor EVPN password EvpnBgp
neighbor UNDERLAY bfd
neighbor UNDERLAY password UnderlayBgp
end
wr
```

Для второго спайна команды те же самые, для лифов поменяется только номер
автономной системы. Также на лифах явно зададим максимальное количество путей
и ECMP маршрутов `maximum-paths 2 ecmp 2`

### Настройка нового лифа

Так как у нас добавился новый лиф для млага, раскатаем на него конфиг, аналогичный
другим лифам:

```text
hostname L4
ip routing
! лупбэки для андерлея и оверлея
interface Loopback0
   ip address 10.0.0.4/32
interface Loopback1
   ip address 10.10.0.4/32
! линки к спайнам
interface Ethernet1
   no switchport
   ip address 172.16.1.7/31
interface Ethernet2
   no switchport
   ip address 172.16.2.7/31
! интерфейсы к клиентам
interface Ethernet3
   switchport mode trunk
interface Ethernet4
   switchport mode trunk
! клиентские вланы
vlan 10
   name VLAN10
vlan 20
   name VLAN20
service routing protocols model multi-agent
vrf instance VRF1
ip routing vrf VRF1
router bgp 65504
   router-id 10.0.0.4
   no bgp default ipv4-unicast
   timers bgp 10 30
   maximum-paths 2 ecmp 2
   neighbor EVPN peer group
   neighbor EVPN remote-as 65500
   neighbor EVPN next-hop-unchanged
   neighbor EVPN update-source Loopback1
   neighbor EVPN bfd
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN password 7 M+hBrrkH3ArRfYYHe6HDng==
   neighbor EVPN send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65500
   neighbor UNDERLAY bfd
   neighbor UNDERLAY password 7 gv6sVndNHh8/e/Z0vgMzFw==
   neighbor 10.10.1.1 peer group EVPN
   neighbor 10.10.2.2 peer group EVPN
   neighbor 172.16.1.6 peer group UNDERLAY
   neighbor 172.16.2.6 peer group UNDERLAY
   !
   vlan 10
      rd auto
      route-target both 10:100010
      redistribute learned
   !
   vlan 20
      rd auto
      route-target both 20:100020
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.0.4/32
      network 10.10.0.4/32
   !
   vrf VRF1
      rd 65504:1
      route-target import evpn 1:111111
      route-target export evpn 1:111111
      redistribute connected
interface Vlan10
   vrf VRF1
   ip address 192.168.1.204/24
   ip virtual-router address 192.168.1.254
interface Vlan20
   vrf VRF1
   ip address 192.168.2.204/24
   ip virtual-router address 192.168.2.254
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 100010
   vxlan vlan 20 vni 100020
   vxlan vrf VRF1 vni 111111
   vxlan learn-restrict any
ip virtual-router mac-address ca:fe:ba:be:00:00
end
wr
```

Также на спайнах поднимем интерфейсы в сторону нового лифа и назначим им IP адрес.

### Подключение клиентов

Новых настроек пока не требуется за исключением коммутации. Клиенты **C1** и
**C2** подключаем к лифам **L1** и **L2**, на которых настроим мультихоминг.
Клиенты **C3** и **C4** подключаем к лифам **L3** и **L4**, на которых настроим
MC-LAG.

## 2. Настройка агрегации в сторону клиентов

На клиентах настроим port-channel в сторону лифов:

```text
enable
conf t
interface Ethernet1
no switchport mode trunk
channel-group 1 mode active
interface Ethernet2
channel-group 1 mode active
interface Port-Channel 1
switchport mode trunk
end
wr
```

Порт-ченнел готов, осталось настроить на лифах:

```text
C1#show port-channel 1
Port Channel Port-Channel1:
  No Active Ports
  Configured, but inactive ports:
       Port         Reason                   
    --------------- -------------------------
       Ethernet1    waiting for LACP response
       Ethernet2    waiting for LACP response
```

## 3. Multihoming ESI LAG

На лифах **L1** и **L2** настроим мультихоминг для клиентов **C1** и **C2**.

Создадим и настроим port-channel на лифах для каждого клиента:

* создадим порт-ченнел и переведём его в транк;
* зададим ESI (последние 4 цифры - номера лифов и номер порт-ченнела);
* укажем RT для импорта Type-4;
* зададим LACP идентификатор;
* добавим физический порт в порт-ченнел.

Пример команд на лифе1:

```text
enable
conf t
interface Port-Channel1
switchport mode trunk
evpn ethernet-segment
  identifier 0000:0000:0000:0000:1201
  route-target import 00:00:00:00:12:01
lacp system-id 0000.0000.1201
interface Ethernet 3
channel-group 1 mode active
end
wr
```

Теперь порт-ченнел в апе (вывод с первого клиента):

```text
C1#show port-channel detail
Port Channel Port-Channel1 (Fallback State: Unconfigured):
Minimum links: unconfigured
Maximum links: unconfigured
Minimum speed: unconfigured
Current weight/Max weight: 2/16
  Active Ports:
     Port         Time Became Active     Protocol     Mode       Weight   State
    ------------ --------------------- ------------ ---------- ---------- -----
     Ethernet1    23:23:13               LACP         Active       1      Rx,Tx
     Ethernet2    23:24:19               LACP         Active       1      Rx,Tx

C1#
```

## 4. Multihoming MC-LAG

Вторую пару лифов (3 и 4) настроим в MLAG пару. Для этого нам потребуется:

* создать служебный Vlan для обмена служебной информацией по пирлинку (только на
  Аристе);
* объединить интерфейсы пирлинка в порт-ченнел;
* указать режим STP point-to-point для пирлинка;
* на интерфейсе keepalive задать IP-адрес;
* создать SVI на служебном Vlan (только на Аристе);
* задать идентификатор MLAG домена;
* указать локальный интерфейс, который будет пирлинком;
* задать адрес соседа;
* указать порт-ченнел пирлинка;
* выбрать keepalive интерфейс;
* настроить отключение всех интерфейсов в случае разрыва связи.

Пример для Leaf3:

```text
enable
conf t
vlan 4094
name MLAG-PEERLINK
trunk group MLAG-PEERLINK
int Ethernet 5
no switchport
ip address 192.168.0.1/30
int Port-Channel 67
switchport mode trunk
switchport trunk group MLAG-PEERLINK
spanning-tree link-type point-to-point
int Ethernet 6
channel-group 67 mode active
int Ethernet 7
channel-group 67 mode active
int vlan 4094
no autostate
ip address 172.16.0.1/30
mlag configuration
domain-id LEAVES-3-4
local-interface Vlan 4094
peer-address 172.16.0.1
peer-link port-channel 67
peer-address heartbeat 192.168.0.2
dual-primary detection delay 1 action errdisable all-interfaces
end
wr
```

Добавим MLAG в конфигурацию BGP. Лиф3 и Лиф4 соединены между собой по iBGP, со
спайнами по eBGP. Изменим номер автономной системы на одинаковый, также уберём
подмену некстхопа в оверлее.

Для работы anycast IP на обоих лифах создадим ещё
один лупбэк с одинаковым адресом и привяжем его к VXLAN.

```text
enable
conf t
interface Loopback2
ip address 10.10.0.103/32
int vx1
no vxlan source-interface Loopback1
vxlan source-interface Loopback2
router bgp 65503
no neighbor EVPN next-hop-unchanged
neighbor UNDERLAY-MLAG peer group
neighbor UNDERLAY-MLAG remote-as 65504
neighbor 172.16.0.2 peer group UNDERLAY-MLAG
address-family ipv4
neighbor UNDERLAY-MLAG activate
network 10.10.0.103/32
end
wr
```

На клиентах настроим портченнелы также, как делали для мультихоминга.

Состояние MLAG:

```text
L3#show mlag
MLAG Configuration:              
domain-id                          :          LEAVES-3-4
local-interface                    :            Vlan4094
peer-address                       :          172.16.0.2
peer-link                          :      Port-Channel67
hb-peer-address                    :         192.168.0.2
peer-config                        :          consistent
                                                       
MLAG Status:                     
state                              :              Active
negotiation status                 :           Connected
peer-link status                   :                  Up
local-int status                   :                  Up
system-id                          :   52:00:00:15:f4:e8
dual-primary detection             :          Configured
dual-primary interface errdisabled :               False
                                                       
MLAG Ports:                      
Disabled                           :                   0
Configured                         :                   0
Inactive                           :                   0
Active-partial                     :                   0
Active-full                        :                   2
```

## Проверка работы

В нормальном состоянии клиенты могут пинговать друг друга. Траффик с C1 идёт через
лиф L1.

```text
C1#traceroute 192.168.2.2
traceroute to 192.168.2.2 (192.168.2.2), 30 hops max, 60 byte packets
 1  192.168.1.201 (192.168.1.201)  9.435 ms  8.976 ms  8.636 ms
 2  * * 192.168.2.2 (192.168.2.2)  10.502 ms
```

Все BGP соседства подняты:

```text
S1#sh ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.1.1, local AS number 65500
Neighbor Status Codes: m - Under maintenance
  Neighbor   V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc PfxAdv
  172.16.1.1 4 65501             84        81    0    0 00:00:32 Estab   2      2      9
  172.16.1.3 4 65502             32        32    0    0 00:00:32 Estab   2      2      9
  172.16.1.5 4 65503             11        11    0    0 00:00:31 Estab   5      5      11
  172.16.1.7 4 65503             11        10    0    0 00:00:31 Estab   5      5      6
```

Сымитируем сбой: отключим для начала лиф L1. Спайны перестали его видеть:

```text
S1#sh ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.1.1, local AS number 65500
Neighbor Status Codes: m - Under maintenance
  Neighbor   V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc PfxAdv
  172.16.1.3 4 65502             40        40    0    0 00:01:32 Estab   2      2      7
  172.16.1.5 4 65503             19        18    0    0 00:01:31 Estab   5      5      9
  172.16.1.7 4 65503             18        17    0    0 00:01:31 Estab   5      5      4
```

Клиент C1 по-прежнему может пинговать соседей (даже в другом VLAN), трафик при
этом идёт через второй лиф (multihoming):

```text
1#traceroute 192.168.2.2
traceroute to 192.168.2.2 (192.168.2.2), 30 hops max, 60 byte packets
 1  * * *
 2  192.168.2.2 (192.168.2.2)  12.300 ms  11.104 ms  12.705 ms
```

Теперь отключим лиф в MLAG паре (L4):

```text
S1#sh ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.1.1, local AS number 65500
Neighbor Status Codes: m - Under maintenance
  Neighbor   V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc PfxAdv
  172.16.1.3 4 65502              8         7    0    0 00:00:14 Estab   2      2      7
  172.16.1.5 4 65503              9         7    0    0 00:00:14 Estab   5      5      4
```

Состояние MLAG:

```text
L3#sh mlag
MLAG Configuration:              
domain-id                          :          LEAVES-3-4
local-interface                    :            Vlan4094
peer-address                       :          172.16.0.2
peer-link                          :      Port-Channel67
hb-peer-address                    :         192.168.0.2
peer-config                        :          consistent
                                                       
MLAG Status:                     
state                              :              Active
negotiation status                 :          Connecting
peer-link status                   :      Lowerlayerdown
local-int status                   :                  Up
system-id                          :   52:00:00:15:f4:e8
dual-primary detection             :             Running
dual-primary interface errdisabled :               False
                                                       
MLAG Ports:                      
Disabled                           :                   0
Configured                         :                   0
Inactive                           :                   0
Active-partial                     :                   2
Active-full                        :                   0
```

Пинг по-прежнему идёт:

```text
C1#ping 192.168.2.2 repeat 1
PING 192.168.2.2 (192.168.2.2) 72(100) bytes of data.
80 bytes from 192.168.2.2: icmp_seq=1 ttl=64 time=7.12 ms

--- 192.168.2.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 7.122/7.122/7.122/0.000 ms
```

Теперь поднимем обратно второй коммутатор в MLAG паре и заставим трафик идти
через пирлинк. На лифе L3 отключим аплинки к спайнам, а на лифе L4 - даунлинки
к клиентам.

```text
L3#sh mlag interfaces 
                                                                   local/remote
  mlag       desc                state       local       remote          status
--------- ---------- -------------------- ----------- ------------ ------------
     1                  active-partial         Po1          Po1         up/down
     2                  active-partial         Po2          Po2         up/down
```

Пинг между клиентами по-прежнему идёт, пакеты видны в дампе wirehark на пирлинке.

## Файлы настроек

Файлы настроек устройств (конфиги) экспортированы в каталог [configs](./configs/).

Готовая лабораторная (экспорт из EVE-NG) - [14_evpn_multihoming.zip](./14_evpn_multihoming.zip).
