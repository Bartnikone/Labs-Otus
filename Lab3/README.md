Настройка ISIS для Underlay-сети.

Цели лабораторной работы:

Настроить ISIS для underlay-сети.

Зафиксировать адресацию, а также четко разграничить роутеры для разных Levels.

Проверить маршртуную информацию.

Убедиться в IP-связности между Loopback-адресами роутеров.

Схема данной лабораторной работы:

![image](https://github.com/user-attachments/assets/c197fef2-7609-4292-ba7a-1737a5871794)

Традиционно, скрин из Eve:

![image](https://github.com/user-attachments/assets/8c028ad7-d21d-4d75-b5ed-bb27dbbcfc15)


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
Итак, Spine_1/2 находятся на Level-1-2 и будут служить связующим звеном для роутеров на Level-1 и Level-2.

Leaf_1/2 будут участниками Level-1. 

Leaf_3 будет находиться в Level-2.

По концепции протокола, участником какой-либо зоны здесь является роутер целиком, а не его линк, как в OSPF.

Также зоны можно разделить на уровни. Участники Level-1 могут общаться лишь друг с другом, для выхода вовне обращаются к роутерам на уровне Level-1-2. 

Роутеры на level-2 имеют глобальное представление о сети, видят все роутеры, участвующие в ISIS.

Также отмечу, что в процессе OSPF была дана команда shutdown.

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

Командой "net 49.0001.0000.0000.000X.00" мы задаем уникальный идентификатор узла в сети, который используется для маршрутизации. 
Он состоит из:
```bash
AFI.AreaID.SystemID.SEL
```
![image](https://github.com/user-attachments/assets/c29890f7-a0f9-49ba-a61d-ac9d2058acb4)

# AFI (Authority and Format Identifier)

Первые два символа (`49`) — это AFI.

Значение `49` зарезервировано для частных (private) сетей (аналогично RFC 1918 в IP).

В публичных сетях используются другие значения (например, `47` для IPv4).

# IDI (Initial Domain Identifier)

Следующие 4 символа (`0001`) — IDI.

Вместе с AFI они формируют Domain ID (в данном случае `49.0001`).

Это идентификатор домена или области (Area) в IS-IS.

# DSP (Domain Specific Part)

Оставшаяся часть (`0000.0000.000X.00`) делится на:

## Area ID (в IS-IS):

`0000` — дополнительная часть идентификатора области (может быть нулевой).

## System ID:

`0000.000X` — уникальный идентификатор узла (роутера) в сети.

Обычно имеет длину 6 байт (в формате `XXXX.XXXX.XXXX`), но в вашем примере используется сокращённая запись.

Часто System ID генерируется из MAC-адреса или назначается вручную.

## SEL (NSAP Selector):

Последние два символа (`00`) — SEL.

Значение `00` указывает, что это адрес узла (роутера), а не сервиса.


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

Замечаем, что наш Leaf_3 видит всех участников ISIS, так как Level-2 является backbone.

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
Добавим на Spine-1 принудительную отдачу все маршрутов из level-2 в level-1:
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
И теперь проверим, что знают наши Leaf_1/2:
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
| I L1       | 10.1.1.13/32  | [115/30]   | 10.0.0.0  | Ethernet1  |

Теперь мы знаем, как пройти к Leaf_3:
```bash
Leaf-1#ping 10.1.1.13
PING 10.1.1.13 (10.1.1.13) 72(100) bytes of data.

80 bytes from 10.1.1.13: icmp_seq=1 ttl=63 time=25.8 ms

80 bytes from 10.1.1.13: icmp_seq=2 ttl=63 time=19.9 ms

80 bytes from 10.1.1.13: icmp_seq=3 ttl=63 time=15.9 ms

80 bytes from 10.1.1.13: icmp_seq=4 ttl=63 time=20.4 ms

80 bytes from 10.1.1.13: icmp_seq=5 ttl=63 time=15.8 ms
```

