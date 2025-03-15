Настройка underlay сети в CLOS топологии из пяти устройств Arista VeOS.
Цели

Собрать топологию Clos из двух Spine И трех Leaf.
Поднять сетевые интерфейсы между Spine и Leaf.
Создать и поднять loopback интерфейсы на всех устройствах.
Распределить адресное пространство для всех физических и loopback-интерфейсов.
Поднять соседство между Spine-Leaf по eBgp, а также передать наши loopback-адреса соседям.
Целевая схема
![image](https://github.com/user-attachments/assets/d4d73e7d-2927-4c47-95f3-e11b390bcd4a)
Адресное пространство для link интерфейсов

10.0.0.0/24

Адресное пространство для loopback интерфейсов

10.1.1.0/24

Типовая конфигурация Spine:
interface Ethernet1
   description low_Leaf1
   no switchport
   ip address 10.0.0.0/31
!
interface Ethernet2
   description low_Leaf2
   no switchport
   ip address 10.0.0.2/31
!
interface Ethernet3
   description low_Leaf3
   no switchport
   ip address 10.0.0.4/31
interface Loopback0
   ip address 10.1.1.1/32
!
ip routing
!
router bgp 65001
   neighbor 10.0.0.1 remote-as 65101
   neighbor 10.0.0.3 remote-as 65102
   neighbor 10.0.0.5 remote-as 65103
   !
   address-family ipv4
      neighbor 10.0.0.1 activate
      neighbor 10.0.0.3 activate
      neighbor 10.0.0.5 activate
      network 10.1.1.1/32

Типовая конфигурация Leaf:
interface Ethernet1
   description Up_Spine1
   no switchport
   ip address 10.0.0.3/31
!
interface Ethernet2
   description Up_Spine2
   no switchport
   ip address 10.0.0.9/31
interface Loopback0
   ip address 10.1.1.12/32
!
ip routing
!
router bgp 65102
   neighbor 10.0.0.2 remote-as 65001
   neighbor 10.0.0.8 remote-as 65002
   !
   address-family ipv4
      neighbor 10.0.0.2 activate
      neighbor 10.0.0.8 activate
      network 10.1.1.12/32

Вывод таблицы маршрутизации с Leaf-1:
Gateway of last resort is not set

 C        10.0.0.0/31 is directly connected, Ethernet1
 C        10.0.0.6/31 is directly connected, Ethernet2
 B E      10.1.1.1/32 [200/0] via 10.0.0.0, Ethernet1
 B E      10.1.1.2/32 [200/0] via 10.0.0.6, Ethernet2
 C        10.1.1.11/32 is directly connected, Loopback0
 B E      10.1.1.12/32 [200/0] via 10.0.0.0, Ethernet1
 B E      10.1.1.13/32 [200/0] via 10.0.0.0, Ethernet1

Как видим, мы получили адреса Loopback каждого роутера по eBgp.

Проверим сетевую связность между Loopback-адресами Leaf-1 - Leaf-3:
Leaf-1#ping 10.1.1.13 source 10.1.1.11
PING 10.1.1.13 (10.1.1.13) from 10.1.1.11 : 72(100) bytes of data.
80 bytes from 10.1.1.13: icmp_seq=1 ttl=63 time=43.0 ms
80 bytes from 10.1.1.13: icmp_seq=2 ttl=63 time=37.7 ms
80 bytes from 10.1.1.13: icmp_seq=3 ttl=63 time=33.0 ms
80 bytes from 10.1.1.13: icmp_seq=4 ttl=63 time=26.1 ms
80 bytes from 10.1.1.13: icmp_seq=5 ttl=63 time=39.1 ms
