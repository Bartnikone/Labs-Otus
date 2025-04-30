# Цель лабораторной работы:

1.Настроить соседство с граничным FW с Leaf-1/3 в разных VRF.
2.Настроить на граничном FW соседство с каждым из leaf.
3.Обеспечить "перетекание" маршрутов из одного vrf в другой на Leaf'ах, путем создания в процессе подъема пиринга на BFW общей таблицы маршрутов.
4.Передать локальный маршрут с Leaf-1 в Vrf lab_8 через BFW обратно на Leaf-1 в другой vrf 2_lab_8.
5.Отдать эти маршруты в evpn vxlan.
6.Проверить доступность адресов по ICMP.
7.Рассмотреть поближе route-type 5 ip-prefix.

### Схема лабораторной работы:

![image](https://github.com/user-attachments/assets/5a6f4c1a-8f55-43a3-8bf5-6fd0864a9a05)

Отличием от прошлой лабораторной является наличие Border Firewall - BFW-1, к которому подключены Leaf-1 и Leaf-3.

### Адресный план для новых префиксов:

```bash

Address Plan:

Leaf-1 - BFW-1

Vrf lab_8

Vlan 40  
10.40.40.1/31  10.40.40.2/31

vlan 21 
21.21.21.21/24 

________________________________________________________

Vrf 2_lab_8

Vlan 20
10.20.20.1/31  10.20.20.2/31

vlan 22
22.22.22.22/24

________________________________________________________


Leaf-3 - BFW-1

Vrf lab_8

Vlan 41
10.41.41.1/30 10.41.41.2/30

Vlan 25
25.25.25.25/24

________________________________________________________


Vrf 2_lab_8

Vlan 42 
10.42.42.1/30 10.42.42.2/30

Vlan 26
26.26.26.26/24
```

### Конфигурация каждого узла:

1) leaf-1

```bash


vlan 20
   name 192.168.20.0/24
!
vlan 21-22,40
!
vrf instance 2_lab_8 - 1-ый подопытный VRF
!
vrf instance L3_Test
!
vrf instance lab_8 - 2-ой подопытный VRF
!
interface Port-Channel1
   mtu 9214
   switchport trunk allowed vlan 10,20,30
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:1111
      route-target import 00:00:00:00:00:11
   lacp system-id 0010.0101.0011
!
interface Port-Channel2
   mtu 9214
   switchport trunk allowed vlan 10,20,30
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:1112
      route-target import 00:00:00:00:00:12
   lacp system-id 0010.0101.0012
!
interface Ethernet1
   description Up_Spine1
   no switchport
   ip address 10.0.0.1/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   isis enable Test
   isis network point-to-point
!
interface Ethernet2
   description Up_Spine2
   no switchport
   ip address 10.0.0.7/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   isis enable Test
   isis network point-to-point
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
   description to_sw-1
   channel-group 1 mode active
!
interface Ethernet6
   description to_sw-2
   channel-group 2 mode active
!
interface Ethernet7
   description to_BFW-1
   switchport trunk allowed vlan 20-22,40
   switchport mode trunk
!
interface Ethernet8
!
interface Loopback0
   ip address 10.1.1.11/32
   ip ospf area 0.0.0.0
   isis enable Test
!
interface Management1
!
interface Vlan10
   vrf L3_Test
   ip address 192.168.10.10/24
!
interface Vlan11
   vrf L3_Test
   ip address 172.16.16.1/24
!
interface Vlan20 - связываем все наши SVI с VRF
   vrf 2_lab_8
   ip address 10.20.20.1/30
!
interface Vlan21
   vrf lab_8
   ip address 21.21.21.21/24
!
interface Vlan22
   vrf 2_lab_8
   ip address 22.22.22.22/24
!
interface Vlan40
   vrf lab_8
   ip address 10.40.40.1/30
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 11 vni 1111
   vxlan vlan 20 vni 2020
   vxlan vlan 30 vni 3030
   vxlan vrf L3_Test vni 666
   vxlan vrf lab_8 vni 777
!
ip routing
ip routing vrf 2_lab_8
ip routing vrf L3_Test
ip routing vrf lab_8
!
ip prefix-list PL_CONNECT seq 10 permit 10.1.1.8/29 le 32
ip prefix-list PL_Overlay seq 11 permit 10.1.1.0/24 le 32
!
route-map RM_Overlay permit 10
   match ip address prefix-list PL_Overlay
!
route-map RM_REDIS_CON permit 10
   match ip address prefix-list PL_CONNECT
!
router bgp 65001
   router-id 10.1.1.11
   no bgp default ipv4-unicast
   maximum-paths 10 ecmp 10
   neighbor EVPN_peers peer group
   neighbor EVPN_peers remote-as 65000
   neighbor EVPN_peers update-source Loopback0
   neighbor EVPN_peers allowas-in 3
   neighbor EVPN_peers ebgp-multihop 3
   neighbor EVPN_peers send-community extended
   neighbor SPINES peer group
   neighbor SPINES remote-as 65000
   neighbor SPINES timers 30 90
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
   vlan 10
      rd 10.1.1.11:10
      route-target both 10:1010
      redistribute learned
   !
   vlan 11
      rd 10.1.1.11:11
      route-target both 11:1111
      redistribute learned
   !
   vlan 20
      rd 10.1.1.11:20
      route-target both 20:2020
      redistribute learned
   !
   vlan 30
      rd 10.1.1.11:30
      route-target both 30:3030
      redistribute learned
   !
   address-family evpn
      neighbor EVPN_peers activate
   !
   address-family ipv4
      neighbor SPINES activate
      network 10.1.1.11/32
   !
   vrf 2_lab_8 - поднимаем соседства с нужных VRF с BFW-1
      rd 10.1.1.11:888
      route-target import evpn 888:1
      route-target export evpn 888:1
      neighbor 10.20.20.2 remote-as 55000
      neighbor 10.20.20.2 update-source Vlan20
      neighbor 10.20.20.2 ebgp-multihop 5
      redistribute connected
      redistribute static
      !
      address-family ipv4
         neighbor 10.20.20.2 activate
   !
   vrf L3_Test
      rd 10.1.1.11:666
      route-target import evpn 666:1
      route-target export evpn 666:1
      redistribute connected
      redistribute static
   !
   vrf lab_2
   !
   vrf lab_8
      rd 10.1.1.11:777
      route-target import evpn 777:1
      route-target export evpn 777:1
      neighbor 10.40.40.2 remote-as 55000
      neighbor 10.40.40.2 update-source Vlan40
      neighbor 10.40.40.2 allowas-in 5
      neighbor 10.40.40.2 ebgp-multihop 5
      redistribute connected
      redistribute static
      !
      address-family ipv4
         neighbor 10.40.40.2 activate
!
router isis Test
   shutdown
   net 49.0001.0000.0000.0003.00
   is-type level-1
   !
   address-family ipv4 unicast
!
router ospf 100
   router-id 3.3.3.3
   shutdown
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   network 10.0.0.0/31 area 0.0.0.0
   network 10.0.0.6/31 area 0.0.0.0
   max-lsa 12000
!
end
Leaf-1#
```

BFW-1

```bash

interface Loopback0
   ip address 1.1.1.1/32
!
interface Management1
!
interface Vlan20
   ip address 10.20.20.2/30
!
interface Vlan40
   ip address 10.40.40.2/30
!
interface Vlan41
   ip address 10.41.41.2/30
!
interface Vlan42
   ip address 10.42.42.2/30
!
interface Vlan50
!
ip routing
ip routing vrf 2_lab_8
ip routing vrf lab_8
!
ip prefix-list PL_to_Leafs seq 10 permit 10.40.40.0/24 le 32
ip prefix-list PL_to_Leafs seq 20 permit 10.20.20.0/24 le 32
!
router bgp 55000
   neighbor 10.20.20.1 remote-as 65001
   neighbor 10.20.20.1 update-source Vlan20
   neighbor 10.20.20.1 ebgp-multihop 5
   neighbor 10.20.20.1 maximum-routes 100
   neighbor 10.40.40.1 remote-as 65001
   neighbor 10.40.40.1 update-source Vlan40
   neighbor 10.40.40.1 ebgp-multihop 5
   neighbor 10.40.40.1 maximum-routes 100
   neighbor 10.41.41.1 remote-as 65003
   neighbor 10.41.41.1 update-source Vlan41
   neighbor 10.41.41.1 ebgp-multihop 5
   neighbor 10.41.41.1 maximum-routes 100
   neighbor 10.42.42.1 remote-as 65003
   neighbor 10.42.42.1 update-source Vlan42
   neighbor 10.42.42.1 ebgp-multihop 5
   neighbor 10.42.42.1 maximum-routes 100
   !
   address-family ipv4
      neighbor 10.20.20.1 activate
      neighbor 10.40.40.1 activate
      neighbor 10.41.41.1 activate
      neighbor 10.42.42.1 activate
!

```

### Важные моменты:

Чтобы мы имели возможность передать маршрут из vrf 2_lab_8 22.22.22.22/24 в vrf lab_8, необходимо обойти защиту от петель eBGP, разрешив принять маршрут со своим номером AS в as-path командой "neighbor 10.40.40.2 allowas-in 5" на Leaf-1 в разделе vrf lab_8. Можно было сделать с помощью override, но на arista не удалось найти подобную команду. Так же создать RM replace as-path не получилось. Поэтому пришлось прибегнуть к разрешению повторения номера AS.

### Теперь проверим таблицу маршрутов на Leaf-1:

```bash

Leaf-1#sh ip route vrf lab_8

VRF: lab_8
 C        10.40.40.0/30 is directly connected, Vlan40
 B E      10.41.41.0/30 [200/0] via VTEP 10.1.1.13 VNI 777 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 C        21.21.21.0/24 is directly connected, Vlan21
 B E      22.22.22.0/24 [200/0] via 10.40.40.2, Vlan40 - маршрут был передан из 2_lab_8 на BFW-1, где затем BFW-1 передал его соседу из vrf lab_8. 
 B E      25.25.25.0/24 [200/0] via 10.40.40.2, Vlan40
 B E      26.26.26.0/24 [200/0] via 10.40.40.2, Vlan40

```

### Затем проверим количество путей для этого маршрута на Leaf-3

```bash
Leaf-3#sh ip bgp 22.22.22.0/24 vrf lab_8
BGP routing table information for VRF lab_8
Router identifier 10.40.40.1, local AS number 65003
BGP routing table entry for 22.22.22.0/24
 Paths: 3 available
  55000 65001
    10.41.41.2 from 10.41.41.2 (1.1.1.1)
      Origin IGP, metric 0, localpref 100, IGP metric 0, weight 0, tag 0
      Received 01:09:47 ago, valid, external, best
      Rx SAFI: Unicast
  65000 65001 55000 65001
    10.1.1.11 from 10.1.1.1 (10.1.1.1), imported EVPN route, RD 10.1.1.11:777
      Origin IGP, metric 0, localpref 100, IGP metric 0, weight 0, tag 0
      Received 00:19:17 ago, valid, external, ECMP head, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:777:1 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d5:5d:c0
      Remote VNI: 777
      Rx SAFI: Unicast
  65000 65001 55000 65001
    10.1.1.11 from 10.1.1.2 (10.1.1.2), imported EVPN route, RD 10.1.1.11:777
      Origin IGP, metric 0, localpref 100, IGP metric 0, weight 0, tag 0
      Received 00:19:16 ago, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:777:1 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d5:5d:c0
      Remote VNI: 777
      Rx SAFI: Unicast
```

Как видно из вывода, у этого маршрута есть 3 пути: Через BFW-1 и через двух spine.

### Проверим теперь получения ip-prefix для этого префикса на Leaf-3:

```bash

 * >Ec    RD: 10.1.1.11:777 ip-prefix 22.22.22.0/24
                                 10.1.1.11             -       100     0       65000 65001 55000 65001 i
 *  ec    RD: 10.1.1.11:777 ip-prefix 22.22.22.0/24
                                 10.1.1.11             -       100     0       65000 65001 55000 65001 i
 * >      RD: 10.1.1.13:777 ip-prefix 22.22.22.0/24
                                 -                     -       100     0       55000 65001 i
```


Теперь посмотри, что передали Spine на Leaf-2:

```bash
 * >Ec    RD: 10.1.1.11:777 ip-prefix 22.22.22.0/24
                                 10.1.1.11             -       100     0       6                                                                                                                                                             5000 65001 55000 65001 i
 *  ec    RD: 10.1.1.11:777 ip-prefix 22.22.22.0/24
                                 10.1.1.11             -       100     0       6                                                                                                                                                             5000 65001 55000 65001 i
 * >Ec    RD: 10.1.1.13:777 ip-prefix 22.22.22.0/24
                                 10.1.1.13             -       100     0       6                                                                                                                                                             5000 65003 55000 65001 i
 *  ec    RD: 10.1.1.13:777 ip-prefix 22.22.22.0/24
                                 10.1.1.13             -       100     0       6                                                                                                                                                  
```

На нем этот префикс мы получили от двух Spine и от двух Leaf.

# И также проверим связность между хостами с Leaf-3 до Leaf-1:

```bash
Leaf-3#ping vrf lab_8 22.22.22.22 source  25.25.25.25
PING 22.22.22.22 (22.22.22.22) from 25.25.25.25 : 72(100) bytes of data.
80 bytes from 22.22.22.22: icmp_seq=1 ttl=63 time=141 ms
80 bytes from 22.22.22.22: icmp_seq=2 ttl=63 time=155 ms
80 bytes from 22.22.22.22: icmp_seq=3 ttl=63 time=168 ms
80 bytes from 22.22.22.22: icmp_seq=4 ttl=63 time=202 ms
80 bytes from 22.22.22.22: icmp_seq=5 ttl=63 time=214 ms
```

### А теперь рассмотрит сообщения bgp с route-type 5:

![image](https://github.com/user-attachments/assets/1357aadb-6b1a-4bdf-bff7-f5a7ac56e69c)

На какие поля нужно обратить внимание в этом дампе:

1) Route Type (5): Указывает, что это маршрут типа IP Prefix.
2) Route Distinguisher (RD):  10.1.1.13:777.
3) ESI (Ethernet Segment Identifier): 00:00:00:00:00:00:00:00:00:00 — нулевой ESI, так как маршрут не связан с multi-homing.

### Таким образом:

Мы смогли передать клиентские префиксы через граничное устройство, а затем добавить их в evpn и распространить по фабрике. Если бы у нас не было связности через Spine, то мы могли бы получать тот ip-prefix только от граничной точки, например, как в сценариях с внешним подключением клиента вне нашего облака. Route Type 5 анонсирует префиксы между VNI через EVPN, используя общий VRF.

