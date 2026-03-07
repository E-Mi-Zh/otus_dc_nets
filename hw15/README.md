# Домашнее задание №8 «VXLAN. Routing»

## Цель

НРеализовать передачу суммарных префиксов через EVPN route-type 5.

* [1. Подключение клиентов.](#1-подключение-клиентов)
* [2. Настройка маршрутизации.](#2-настройка-маршрутизации)

## Топология

Топология лабораторного стенда собрана в среде EVE-NG.

![Топология с IP адресами](topology.drawio.png)

## 1. Подключение клиентов

В качестве основы возьмём предыдущую лабораторную работу по L3 EVPN multihoming,
таким образом часть настроек уже выполнена и останется лишь внести изменения.

Переместим VLAN 20 в новый VRF. Для этого на лифах:

* создадим новый VRF;
* переместим VLAN20 в новый VRF;
* для нового VRF добавим VNI;
* добавим VRF2 в BGP.

Пример команд для первого лифа:

```text
enable
conf t
vrf instance VRF2
ip routing vrf VRF2
int vlan 20
no vrf VRF1
vrf VRF2
  ip address 192.168.2.202/24
  ip virtual-router address 192.168.2.254
int vxlan 1
  vxlan vrf VRF2 vni 222222
router bgp 65501
vrf VRF2
  rd 65501:2
  route-target import evpn 2:222222
  route-target export evpn 2:222222
  redistribute connected
end
wr
```

Для второго лифа команды те же самые, поменяется только номер
автономной системы.

Информация о VRF:

```text
L1#sh vrf
Maximum number of VRFs allowed: 1024
   VRF           Protocols       State         Interfaces        
------------- --------------- ---------------- ------------------
   VRF1          IPv4            routing       Vl10, Vl4097      
   VRF1          IPv6            no routing    Vl4097            
   VRF2          IPv4            routing       Vl20, Vl4098      
   VRF2          IPv6            no routing    Vl4098            
   default       IPv4            routing       Et1, Et2, Lo0, Lo1
   default       IPv6            no routing      
```

Новых настроек на клиентах не требуется.

## 2. Настройка маршрутизации

Настроим клиент С3 в качестве маршрутизатора для остальных. Он будет осуществлять
импорт из VRF и анонс суммарных маршрутов.

* добавим врфы, соответствующие вланы и адреса в них;
* создадим служебный влан в GRT;
* добавим лупбэки для андерлей и оверлей;
* сконфигурируем BGP (eBGP пиринг с лифами L3 и L$).

```text
enable
conf t
vrf instance VRF1
ip routing vrf VRF1
vrf instance VRF2
ip routing vrf VRF2
ip routing
interface Vlan10
   vrf VRF1
   ip address 192.168.1.253/24
interface Vlan20
   vrf VRF2
   ip address 192.168.2.253/24
vlan 100
int vlan 100
ip address 172.16.100.2/29
interface Loopback0
   ip address 10.0.0.100/32
interface Loopback1
   ip address 10.10.0.100/32
router bgp 65534
  router-id 10.0.0.100
  no bgp default ipv4-unicast
  timers bgp 10 30
  maximum-paths 2 ecmp 2
  neighbor EVPN peer group
  neighbor EVPN remote-as 65503
  neighbor EVPN update-source Loopback1
  neighbor EVPN bfd
  neighbor EVPN ebgp-multihop 3
  neighbor EVPN password 7 M+hBrrkH3ArRfYYHe6HDng==
  neighbor EVPN send-community extended
  neighbor UNDERLAY peer group
  neighbor UNDERLAY remote-as 65503
  neighbor UNDERLAY bfd
  neighbor UNDERLAY password 7 gv6sVndNHh8/e/Z0vgMzFw==
  neighbor 10.10.0.3 peer group EVPN
  neighbor 10.10.0.4 peer group EVPN
  neighbor 172.16.100.3 peer group UNDERLAY
  neighbor 172.16.100.4 peer group UNDERLAY
address-family ipv4
  neighbor UNDERLAY activate
  network 10.0.0.100/32
  network 10.10.0.100/32
address-family evpn
  neighbor EVPN activate
vrf VRF1
  rd 65534:1
  route-target import evpn 1:111111
  route-target import evpn 2:222222
  route-target export evpn 1:111111
  redistribute static
vrf VRF2
  rd 65534:2
  route-target import evpn 2:222222
  route-target import evpn 1:111111
  route-target export evpn 2:222222
  redistribute static
end
wr
```

На лифах L1 и L2 также создадим служебный влан, назначим адреса в нём, добавим
eBGP соседство с C3:

```text
enable
conf t
vlan 100
int vlan 100
ip address 172.16.100.4/29
router bgp 65503
   neighbor 172.16.100.2 remote-as 65534
   neighbor 172.16.100.2 bfd
   address-family ipv4
      neighbor 172.16.100.2 activate
      network 10.10.0.100/32
vrf VRF2
      rd 65503:12
      route-target import evpn 2:222222
      route-target export evpn 2:222222
      redistribute connected
end
wr
```

BGP соседство поднялось:

```text
C3#sh bgp evpn summary 
BGP summary information for VRF default
Router identifier 10.0.0.100, local AS number 65534
Neighbor Status Codes: m - Under maintenance
  Neighbor  V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc PfxAdv
  10.10.0.3 4 65503              0         0    0    0 01:26:55 Connect
  10.10.0.4 4 65503              0         0    0    0 01:26:55 Connect
C3#sh bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.100, local AS number 65534
Neighbor              AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
------------ ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.10.0.3          65503 Connect       IPv4 Unicast            Configured              0          0          0
10.10.0.3          65503 Connect       L2VPN EVPN              Configured              0          0          0
10.10.0.4          65503 Connect       IPv4 Unicast            Configured              0          0          0
10.10.0.4          65503 Connect       L2VPN EVPN              Configured              0          0          0
172.16.100.3       65503 Established   IPv4 Unicast            Negotiated             13         13          7
172.16.100.4       65503 Established   IPv4 Unicast            Negotiated             13         13         10
C3#sh ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.0.100, local AS number 65534
Neighbor Status Codes: m - Under maintenance
  Neighbor     V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc PfxAdv
  10.10.0.3    4 65503              0         0    0    0 01:27:06 Connect
  10.10.0.4    4 65503              0         0    0    0 01:27:06 Connect
  172.16.100.3 4 65503            672       624    0    0 00:08:38 Estab   13     13     7
  172.16.100.4 4 65503            664       630    0    0 00:08:38 Estab   13     13     10
```

Однако маршруты не экспортируются, и, соответственно, пинг между VRF не идёт.

## Файлы настроек

Файлы настроек устройств (конфиги) экспортированы в каталог [configs](./configs/).

Готовая лабораторная (экспорт из EVE-NG) - [15_vxlan_routing.zip](./15_vxlan_routing.zip).
