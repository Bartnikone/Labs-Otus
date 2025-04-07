### Цель лабораторной работы:
1. Поднять соседство по eBGP между Spine/Leaf в AF ipv4.
2. Проаносировать через эти соседства адреса Lo0.
3. Поднять eBGP между Lo0 в AF evpn.
4. Настроить bgp evpn vxlan.
5. Проверить связность между устройствами по l2vpn.

Итак, схема лабораторных работ остается почти неизменной:

![image](https://github.com/user-attachments/assets/4dd96df8-f779-45a2-a29f-4d66149a5157)

Для начала нам необходимо настроить eBGP пиринг и проанонсировать Lo0:

*Сразу небольшая ремарочка, чтобы на Arista можно было поднимать соседство по BGP в отличных от ipv4 AF, нужно дать команду :"service routing protocols model multi-agent" и отправить устройства в ребут.*

# Пример конфигурации со Spine-1:

```bash
ip prefix-list PL_Overlay seq 11 permit 10.1.1.0/24 le 32 - разрешим для удобства тестирования всю /24 подсеть.
!
route-map RM_Overlay permit 10
   match ip address prefix-list PL_Overlay 
!
router bgp 65000
   router-id 10.1.1.1
   no bgp default ipv4-unicast
   maximum-paths 10 ecmp 10
   neighbor EVPN_peers peer group 
   neighbor EVPN_peers remote-as 65001
   neighbor EVPN_peers next-hop-unchanged 
   neighbor EVPN_peers update-source Loopback0
   neighbor EVPN_peers ebgp-multihop 3
   neighbor EVPN_peers send-community extended
   neighbor LEAFS peer group
   neighbor LEAFS bfd
   neighbor LEAFS timers 3 9
   neighbor LEAFS route-map RM_Overlay in
   neighbor LEAFS route-map RM_Overlay out
   neighbor LEAFS maximum-routes 1000
   neighbor 10.0.0.1 peer group LEAFS
   neighbor 10.0.0.1 remote-as 65001
   neighbor 10.0.0.3 peer group LEAFS
   neighbor 10.0.0.3 remote-as 65002
   neighbor 10.0.0.5 peer group LEAFS
   neighbor 10.0.0.5 remote-as 65003
   neighbor 10.1.1.11 peer group EVPN_peers
   neighbor 10.1.1.11 remote-as 65001
   neighbor 10.1.1.12 peer group EVPN_peers
   neighbor 10.1.1.12 remote-as 65002
   no neighbor 10.1.1.12 shutdown
   neighbor 10.1.1.13 peer group EVPN_peers
   neighbor 10.1.1.13 remote-as 65003
   !
   address-family evpn
      neighbor EVPN_peers activate
   !
   address-family ipv4
      neighbor LEAFS activate
      network 10.1.1.1/32
!

```

# Разберем основные команды:

*neighbor EVPN_peers next-hop-unchanged - необходимый параметр, чтобы построить vxlan туннель между Lo0 Leaf'ов, нужно, чтобы Spine'ы не подменивали NH на свой собственный адрес, а передавали его без изменения. Это сработает, так как в AF ipv4 мы уже передали адреса Lo0 каждого Leaf, то есть NH будет доступен.*

*__neighbor EVPN_peers update-source Loopback0__ - источник построения соседства eBGP - Lo0.*

*neighbor EVPN_peers ebgp-multihop 3 - разрешает установку eBGP-сессий через 3 хопа. Так как мы строит сессии с Lo0, необходимый пункт.*

*neighbor EVPN_peers send-community extended - позволяет EVPN передавать Route Target (RT) и Route Distinguisher (RD).*

# И пример конфигурации с Leaf-1:

```bash
ip prefix-list PL_Overlay seq 11 permit 10.1.1.0/24 le 32
!
route-map RM_Overlay permit 10
   match ip address prefix-list PL_Overlay
!
router bgp 65001
   router-id 10.1.1.11
   no bgp default ipv4-unicast
   maximum-paths 10 ecmp 10
   neighbor EVPN_peers peer group
   neighbor EVPN_peers remote-as 65000
   neighbor EVPN_peers update-source Loopback0
   neighbor EVPN_peers ebgp-multihop 3
   neighbor EVPN_peers send-community extended
   neighbor SPINES peer group
   neighbor SPINES remote-as 65000
   neighbor SPINES bfd
   neighbor SPINES timers 3 9
   neighbor SPINES maximum-routes 1000
   neighbor 10.0.0.0 peer group SPINES
   no neighbor 10.0.0.0 route-map in
   no neighbor 10.0.0.0 route-map out
   neighbor 10.0.0.6 peer group SPINES
   neighbor 10.1.1.1 peer group EVPN_peers
   neighbor 10.1.1.1 remote-as 65000
   neighbor 10.1.1.2 peer group EVPN_peers
   neighbor 10.1.1.2 remote-as 65000
   redistribute connected route-map RM_Overlay
   !
 address-family evpn
      neighbor EVPN_peers activate
   !
   address-family ipv4
      neighbor SPINES activate
      network 10.1.1.11/32

```

# Отлично, соседство по BGP в AF EVPN мы настроили, проверим, что все поднялось.
Таблица BGP EVPN-соседей:
```bash
Leaf-1#sh bgp evpn summary
Neighbor	V	AS	MsgRcvd	MsgSent	InQ	OutQ	Up/Down	State	PfxRcd	PfxAcc
10.1.1.1	4	65000	110	105	0	0	01:12:53	Estab	2	2
10.1.1.2	4	65000	41	37	0	0	00:18:56	Estab	2	2
```

# Затем нам необходимо сконфигурировать адресное пространство для наших клиентов:
```bash
PC1 - 192.168.0.2/24
PC2 - 192.168.0.3/24
PC3 - 192.168.0.4/24
```
Подключаем каждого клиента к eth3 каждого Leaf.

# Далее прикладываю типовую настройку Leaf:
```bash
interface Ethernet3
   description to_PC1
   switchport access vlan 10
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
!
router bgp 65001
 vlan 10
      rd 10.1.1.11:10
      route-target both 10:1010
      redistribute learned
```
# Рассмотрим по порядку, что мы настроили:
1. Настроили vlan на каждом порту в сторону клиентов.
2. Настроили int vxlan1.
vxlan source-interface Loopback0 - адрес VTEP'а будет наш Lo0.
vxlan udp-port 4789 - дефолтное значение, оно автоматически подставляет этот порт, он должен быть одинаковым для всех наших VTEP.
vxlan vlan 10 vni 1010 - связываем VLAN 10 с VXLAN VNI 1010, это позволяет трафику из VLAN 10 передаваться через VXLAN-туннель с идентификатором vni 1010.
3. Настроили BGP EVPN для нашего Vlan 10.
rd 10.1.1.11:10 - задаем вручную уникальный идентификатор для маршрутов EVPN в данном VLAN.
route-target both 10:1010 - определяет, какие маршруты EVPN импортируем/экспортируем, both - RT применяется и для импорта, и для экспорта.
redistribute learned - разрешаем передачу в EVPN локально изученных MAC-адресов из VLAN 10, так другие VTEP узнают, какие mac'и живут за нашим VTEP.

# Теперь проверим, есть ли связность между PC1 и PC2:
```bash
VPCS> ping 192.168.0.3

84 bytes from 192.168.0.3 icmp_seq=1 ttl=64 time=183.058 ms
84 bytes from 192.168.0.3 icmp_seq=2 ttl=64 time=61.888 ms
84 bytes from 192.168.0.3 icmp_seq=3 ttl=64 time=39.848 ms
84 bytes from 192.168.0.3 icmp_seq=4 ttl=64 time=39.583 ms
```
И посмотрим на вывод таблицы evpn на Leaf-1:
```bash

Leaf-1#sh bgp evpn route-type mac-ip vni 1010
BGP routing table information for VRF default
Router identifier 10.1.1.11, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.1.11:10 mac-ip 0050.7966.680a
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.1.12:10 mac-ip 0050.7966.680b
                                 10.1.1.12             -       100     0       65000 65002 i
 *  ec    RD: 10.1.1.12:10 mac-ip 0050.7966.680b
                                 10.1.1.12             -       100     0       65000 65002 i
 * >Ec    RD: 10.1.1.13:10 mac-ip 0050.7966.680c
                                 10.1.1.13             -       100     0       65000 65003 i
 *  ec    RD: 10.1.1.13:10 mac-ip 0050.7966.680c
                                 10.1.1.13             -       100     0       65000 65003 i
```

Видим, что mac'и изучены от других VTEP.

# Проверим маршруты до самих VTEP:

```bash

Leaf-1#sh bgp evpn route-type imet vni 1010
BGP routing table information for VRF default
Router identifier 10.1.1.11, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.1.11:10 imet 10.1.1.11
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.1.12:10 imet 10.1.1.12
                                 10.1.1.12             -       100     0       65000 65002 i
 *  ec    RD: 10.1.1.12:10 imet 10.1.1.12
                                 10.1.1.12             -       100     0       65000 65002 i
 * >Ec    RD: 10.1.1.13:10 imet 10.1.1.13
                                 10.1.1.13             -       100     0       65000 65003 i
 *  ec    RD: 10.1.1.13:10 imet 10.1.1.13
                                 10.1.1.13             -       100     0       65000 65003 i
```
 
# И прикладываю скрин update-сообщения BGP, в котором мы получаем mac-адрес PC2:

![image](https://github.com/user-attachments/assets/82441242-d0fc-4b7e-8209-9bec0e2e525a)

Важные поля в нем:
```bash
Address family identifier (AFI): Layer-2 VPN (25) 
Subsequent address family identifier (SAFI): EVPN (70) 
Route Type: MAC Advertisement Route (2) 
Route Distinguisher: 00010a01010c000a (10.1.1.12:10)
Route Target: 10:1010 [Transitive 2-Octet AS-Specific]
Encapsulation: VXLAN Encapsulation [Transitive Opaque]
```
