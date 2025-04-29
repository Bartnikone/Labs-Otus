### Цель лабораторной работы:

1.Подключить PC к коммутатору.

2.Коммутатора в свою очередь подключить к Leaf.

3.Со стороны коммутаторов собрать LACP в сторону двух Leaf.

4.Со стороны Leaf собрать LACP в сторону коммутаторов, объединить Port-channel в свой evpn ethernet-segment.

5.Проверить отказоустойчивость - убедиться, что связнность не теряется при отключении одного из линков.

# Прикладываю схему для этого работы:

![image](https://github.com/user-attachments/assets/8e182a54-ddd7-422a-8442-a5872684bbcf)

Типовая настройка коммутатора на примере Sw-1:

```bash
vlan 10
   name 192.168.10.0/24
!
vlan 20
   name 192.168.20.0/24
!
vlan 30
   name 192.168.30.0/24
!
interface Port-Channel1
   description to_Leaf-1-2
   mtu 9214
   switchport trunk allowed vlan 10,20,30
   switchport mode trunk
   lacp system-id 0010.0202.0001 - для удобства идентификации SW-1-2 на Leaf-1-2 задал и для коммутаторов sys-id, конфигурация id на основе Lo0
!
interface Ethernet1
   description tp_leaf-1
   channel-group 1 mode active - помещаем интерфейсы в сторону Leaf-1 в PO 1
!
interface Ethernet2
   description tp_leaf-2
   channel-group 1 mode active
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
   description to_PC1
   switchport access vlan 10
!
interface Ethernet7
   description to_PC2
   switchport access vlan 20
!
interface Ethernet8
   description to_PC3
   switchport access vlan 30
!
interface Loopback0
   ip address 10.2.2.1/32
```

Ну других SW настройки будут аналогичными. Далее прикладываю настройки на Leaf-1:

```bash
vlan 10
   name 192.168.10.0/24
!
vlan 11
   name L3_Test
!
vlan 20
   name 192.168.20.0/24
!
vlan 30
   name 192.168.30.0/24
!
vrf instance L3_Test
!
interface Port-Channel1 - создаем PO1 для линка в сторону Sw-1
   mtu 9214
   switchport trunk allowed vlan 10,20,30
   switchport mode trunk
   !
   evpn ethernet-segment - задаем параметры ES для SW-1, грубо говоря, помещаем линк в один единый домен
      identifier 0000:0000:0000:0000:1111
      route-target import 00:00:00:00:00:11
   lacp system-id 0010.0101.0011
!
interface Port-Channel2 - создаем PO1 для линка в сторону Sw-2
   mtu 9214
   switchport trunk allowed vlan 10,20,30
   switchport mode trunk
   !
   evpn ethernet-segment -  задаем параметры ES для SW-1
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
interface Ethernet5 - привязываем порты в Sw-1
   description to_sw-1
   channel-group 1 mode active
!
interface Ethernet6  - привязываем порты в Sw-2
   description to_sw-2
   channel-group 2 mode active
!
interface Ethernet7
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
interface Vlan10 - добавляем int vlan10 для возможности маршуртизироваться через l3vni
   vrf L3_Test
   ip address 192.168.10.10/24
!
interface Vlan11
   vrf L3_Test
   ip address 172.16.16.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010 - объявляем набор vni для наших vlan
   vxlan vlan 11 vni 1111
   vxlan vlan 20 vni 2020
   vxlan vlan 30 vni 3030
   vxlan vrf L3_Test vni 666
!
ip routing
ip routing vrf L3_Test
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
   neighbor SPINES bfd
   neighbor SPINES timers 30 90 - большой счетчик), но иначе в виртуализации начинаются беды с BGP.
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
   vlan 10 - создаем mac-vrf
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
   vrf L3_Test создаем ip-vrf
      rd 10.1.1.11:666
      route-target import evpn 666:1
      route-target export evpn 666:1
      redistribute connected
      redistribute static
```

Проверка состояния соседей в LACP:

![image](https://github.com/user-attachments/assets/c4494c9e-5970-4294-99a7-d81a929615d2)


# Теперь проверим, какие сообщения будут присланы на Leaf-3, если мы включим Po1/2 на Leaf-1:

1)Итак, первый тип сообщений evpn - Ethernet Auto-Discovery (type 1), он используется для объявления доступности ES и управления DF.

![image](https://github.com/user-attachments/assets/62d76799-94fa-4ce7-8e9b-6c524277f26c)

2)И второй тип сообщения evpn для работы с Ethernet Segment - Ethernet Segment (type 4), с помощью него возможна синхронизация информации между PE-устройствами в Multi-Homing.

![image](https://github.com/user-attachments/assets/e5468465-f34d-4d3c-a272-c649b1d2674f)

# Осуществим проверку между l2vni:

![image](https://github.com/user-attachments/assets/bf844eab-3b9f-4e9f-a989-002578e40a61)

```bash
VPCS> ping 192.168.10.13

84 bytes from 192.168.10.13 icmp_seq=1 ttl=64 time=853.644 ms
84 bytes from 192.168.10.13 icmp_seq=2 ttl=64 time=121.399 ms
84 bytes from 192.168.10.13 icmp_seq=3 ttl=64 time=220.378 ms
84 bytes from 192.168.10.13 icmp_seq=4 ttl=64 time=97.565 ms
84 bytes from 192.168.10.13 icmp_seq=5 ttl=64 time=137.430 ms
```

И через l3vni:

![image](https://github.com/user-attachments/assets/b5455617-711a-4209-be96-8e9d6abca2b9)

```bash
VPCS> ping 192.168.30.33

84 bytes from 192.168.30.33 icmp_seq=1 ttl=62 time=129.862 ms
84 bytes from 192.168.30.33 icmp_seq=2 ttl=62 time=159.687 ms
84 bytes from 192.168.30.33 icmp_seq=3 ttl=62 time=267.316 ms
84 bytes from 192.168.30.33 icmp_seq=4 ttl=62 time=167.390 ms
84 bytes from 192.168.30.33 icmp_seq=5 ttl=62 time=118.486 ms
```

Проверки увенчались успехом.

Теперь я отключу один линк в сторону leaf-2 от sw-1:

![image](https://github.com/user-attachments/assets/8b5507e1-449f-4fdc-8a99-48821431903f)

![image](https://github.com/user-attachments/assets/9552dfb0-dd3b-4038-9575-778b863311d3)

```bash
VPCS> ping 192.168.10.13

84 bytes from 192.168.10.13 icmp_seq=1 ttl=64 time=88.794 ms
84 bytes from 192.168.10.13 icmp_seq=2 ttl=64 time=287.793 ms
84 bytes from 192.168.10.13 icmp_seq=3 ttl=64 time=121.459 ms
84 bytes from 192.168.10.13 icmp_seq=4 ttl=64 time=227.837 ms
84 bytes from 192.168.10.13 icmp_seq=5 ttl=64 time=101.557 ms
```
Как видим, связность не потерялась, трафик пошел лишь в сторону Leaf-1.
