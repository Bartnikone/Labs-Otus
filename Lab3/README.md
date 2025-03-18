В процессе...
Схема данной лабораторной работы:
![image](https://github.com/user-attachments/assets/c197fef2-7609-4292-ba7a-1737a5871794)

```bash
Сразу небольшая ремарочка:
Столкнулся с проблемой, что виртуальная Ариста отнимает 3 байта от MTU на интерфейсе. У меня был указан везде MTU 9214, при этом вывод isis interface давал значение в 9211:

Spine-1#sh isis Test interface

IS-IS Instance: Test VRF: default

  Interface Ethernet1:
    Index: 14 SNPA: P2P
    MTU: 9211 Type: point-to-point

При этом MTU на обоих Spine/Leaf были одинаковые с точки зрения isis - 9211. Но соседство не хотело подниматься. 

Пришлось убрать MTU с интерфейсов, только тогда соседство поднялось:
### IS-IS Neighbors for Instance Test on Leaf-1

| Instance | VRF     | System Id | Type | Interface | SNPA | State | Hold Time | Circuit Id |
|----------|---------|-----------|------|-----------|------|-------|-----------|------------|
| Test     | default | Spine-1   | L1   | Ethernet1 | P2P  | UP    | 27        | 0E         |
```
Итак, Spine_1/2 являются участника зоны Level-1-2 и будут служить связующим звеном для роутеров Level-1 и Level-2.

Leaf_1/2 будут участника Level-1. 

Leaf_3 будет находиться в Level-2.

Пример конфигурации для Spine-роутеров:
```bash
interface Ethernet1
   description low_Leaf1
   no switchport
   ip address 10.0.0.0/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   isis enable Test
   no isis hello padding
   isis network point-to-point
!
interface Ethernet2
   description low_Leaf2
   no switchport
   ip address 10.0.0.2/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.1
   isis enable Test
   isis network point-to-point
!
interface Ethernet3
   description low_Leaf3
   no switchport
   ip address 10.0.0.4/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.3
   isis enable Test
   isis network point-to-point
!
interface Loopback0
   ip address 10.1.1.1/32
   ip ospf area 0.0.0.0
   isis enable Test
!
route-map Test_ISIS permit 10
   match ip address prefix-list ALL
!
router isis Test
   net 49.0001.0000.0000.0001.00
   log-adjacency-changes
   !
   address-family ipv4 unicast
!

```

Пример конфигуарции для Leaf:
```bash
interface Ethernet1
   description Up_Spine1
   no switchport
   ip address 10.0.0.3/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.1
   isis enable Test
   isis network point-to-point
!
interface Ethernet2
   description Up_Spine2
   no switchport
   ip address 10.0.0.9/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.1
   isis enable Test
   isis network point-to-point
!
interface Loopback0
   ip address 10.1.1.12/32
   ip ospf area 0.0.0.1
   isis enable Test
!
router isis Test
   net 49.0001.0000.0000.0004.00
   is-type level-1
   !
   address-family ipv4 unicast
!

```
Для Leaf_3 в процессе ISIS изменится is-type на level-2.

Итак, проверим, что все соседства поднялись на Spine-1:
### IS-IS Neighbors for Instance Test on Spine-1

| Instance | VRF     | System Id | Type | Interface | SNPA | State | Hold Time | Circuit Id |
|----------|---------|-----------|------|-----------|------|-------|-----------|------------|
| Test     | default | Leaf-1    | L1   | Ethernet1 | P2P  | UP    | 27        | 0E         |
| Test     | default | Leaf-2    | L1   | Ethernet2 | P2P  | UP    | 28        | 0E         |
| Test     | default | Leaf-3    | L2   | Ethernet3 | P2P  | UP    | 28        | 0E         |

И на Leaf_3:
### IS-IS Neighbors for Instance Test on Leaf-3

| Instance | VRF     | System Id | Type | Interface | SNPA | State | Hold Time | Circuit Id |
|----------|---------|-----------|------|-----------|------|-------|-----------|------------|
| Test     | default | Spine-1   | L2   | Ethernet1 | P2P  | UP    | 26        | 10         |
| Test     | default | Spine-2   | L2   | Ethernet2 | P2P  | UP    | 29        | 10         |

Самое время проверить маршруты, которые мы видим. Например, на Leaf_3:

| Route Type | Destination   | Metric     | Next Hop  | Interface  |
|------------|---------------|------------|-----------|------------|
| S          | 8.8.8.8/32    | -          | -         | Null0      |
| I L2       | 10.0.0.0/31   | [115/20]   | 10.0.0.4  | Ethernet1  |
| I L2       | 10.0.0.2/31   | [115/20]   | 10.0.0.4  | Ethernet1  |
| C          | 10.0.0.4/31   | -          | -         | Ethernet1  |
| I L2       | 10.0.0.6/31   | [115/20]   | 10.0.0.10 | Ethernet2  |
| I L2       | 10.0.0.8/31   | [115/20]   | 10.0.0.10 | Ethernet2  |
| C          | 10.0.0.10/31  | -          | -         | Ethernet2  |
| I L2       | 10.1.1.1/32   | [115/20]   | 10.0.0.4  | Ethernet1  |
| I L2       | 10.1.1.2/32   | [115/20]   | 10.0.0.10 | Ethernet2  |
| I L2       | 10.1.1.11/32  | [115/30]   | 10.0.0.4  | Ethernet1  |
|            |               |            | 10.0.0.10 | Ethernet2  |
| I L2       | 10.1.1.12/32  | [115/30]   | 10.0.0.4  | Ethernet1  |
|            |               |            | 10.0.0.10 | Ethernet2  |
| C          | 10.1.1.13/32  | -          | -         | Loopback0  |

Замечаем, что наш Leaf_3 видит всех участников ISIS, так как Level-2 зона является backbone.

Перейдем в таблицу маршрутов на Leaf_1:
### Routing Table

| Route Type | Destination   | Metric     | Next Hop  | Interface  |
|------------|---------------|------------|-----------|------------|
| C          | 10.0.0.0/31   | -          | -         | Ethernet1  |
| I L1       | 10.0.0.2/31   | [115/20]   | 10.0.0.0  | Ethernet1  |
| I L1       | 10.0.0.4/31   | [115/20]   | 10.0.0.0  | Ethernet1  |
| C          | 10.0.0.6/31   | -          | -         | Ethernet2  |
| I L1       | 10.0.0.8/31   | [115/20]   | 10.0.0.6  | Ethernet2  |
| I L1       | 10.0.0.10/31  | [115/20]   | 10.0.0.6  | Ethernet2  |
| I L1       | 10.1.1.1/32   | [115/20]   | 10.0.0.0  | Ethernet1  |
| I L1       | 10.1.1.2/32   | [115/20]   | 10.0.0.6  | Ethernet2  |
| C          | 10.1.1.11/32  | -          | -         | Loopback0  |
| I L1       | 10.1.1.12/32  | [115/30]   | 10.0.0.0  | Ethernet1  |
|            |               |            | 10.0.0.6  | Ethernet2  |
И мы не видим здесь Leaf_3. Потому что по умолчанию маршруты из Level-2 не передаются в Level-1. Но мы можем это исправить. Для начала проверим связность между Leaf_3/1:
```bash
Leaf-3#ping 10.1.1.11

PING 10.1.1.11 (10.1.1.11) 72(100) bytes of data.

80 bytes from 10.1.1.11: icmp_seq=1 ttl=63 time=31.1 ms

80 bytes from 10.1.1.11: icmp_seq=2 ttl=63 time=24.3 ms

80 bytes from 10.1.1.11: icmp_seq=3 ttl=63 time=28.7 ms

80 bytes from 10.1.1.11: icmp_seq=4 ttl=63 time=16.5 ms

80 bytes from 10.1.1.11: icmp_seq=5 ttl=63 time=19.0 ms
```

И в обратную сторону:
```bash
Leaf-1#ping 10.1.1.13

PING 10.1.1.13 (10.1.1.13) 72(100) bytes of data.

From 10.0.0.1 icmp_seq=1 Destination Host Unreachable

From 10.0.0.1 icmp_seq=2 Destination Host Unreachable

From 10.0.0.1 icmp_seq=3 Destination Host Unreachable

From 10.0.0.1 icmp_seq=4 Destination Host Unreachable

From 10.0.0.1 icmp_seq=5 Destination Host Unreachable
```
Ну конечно, у нас же нет даже дефолтного маршрута в сторону наших связующих звеньев - Spine1/2. 

Давайте это изменим:
Добавим на Spine-1 принудительную отдачу все маршрутов из level-2 в level-1(можно и отдать просто дефолт в сторону Level-1):
```bash
Spine-1(config)#router isis Test

Spine-1(config-router-isis)#address-family ipv4

Spine-1(config-router-isis-af)#redistribute isis level-2 into level-1 route-map
 Isis_RM
```
И префикс-лист:
```bash
Spine-1(config)#route-map Test_ISIS
Spine-1(config-route-map-Test_ISIS)# match ip address prefix-list ALL

Spine-1(config-route-map-Test_ISIS)#exit

Spine-1(config)#ip prefix-list ALL seq 5 permit 0.0.0.0/0 le 32
```


