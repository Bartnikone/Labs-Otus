Настройка OSPF для Underlay-сети.

Цели лабораторной работы:
1. Настроить OSPF для разных Area.
2. Зафиксировать адресацию, а также четко разграничить линки для разных Area.
3. Проверить Database на Spine/Leaf
4. Убедиться в IP-связности между Loopback-адресами роутеров.

Позвольте прибегнуть к скриншотам и типовым схемам. 
Разграничения на зоны представлено на скриншоте ниже:
![image](https://github.com/user-attachments/assets/0314e13e-0a7e-4c50-9199-4186ce6eb0b2)

Фактически в Eve это выглядит следующим образом:
![image](https://github.com/user-attachments/assets/371f68e0-29e6-4772-b3a0-c50638db6da1)

Для удобства читаемости схемы и сверки ее с Database на устройствах также прилагаю схему адресации:
![image](https://github.com/user-attachments/assets/bd137037-b50d-4624-a95e-90f6cf15eda6)

Прикладываю типовую конфигурацию для Spine и Leaf, на остальных устройствах эта конфигурация будет аналогичная за исключением верного указания зон.
Самое интересное, на мой взгляд, настройка зоны NSSA, поэтому приложу конфигурацию для нее.
Spine_1:
```bash
interface Ethernet1
   description low_Leaf1
   mtu 9214
   no switchport
   ip address 10.0.0.0/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet2
   description low_Leaf2
   mtu 9214
   no switchport
   ip address 10.0.0.2/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.1
!
interface Ethernet3
   description low_Leaf3
   mtu 9214
   no switchport
   ip address 10.0.0.4/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.3
!
interface Loopback0
   ip address 10.1.1.1/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 100
   router-id 1.1.1.1
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   area 0.0.0.3 nssa
   network 10.0.0.0/31 area 0.0.0.0
   network 10.0.0.2/31 area 0.0.0.1
   network 10.0.0.4/31 area 0.0.0.3
   max-lsa 12000
```
Leaf_3:
```bash
interface Ethernet1
   description Up_Spine1
   mtu 9214
   no switchport
   ip address 10.0.0.5/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.3
!
interface Ethernet2
   description Up_Spine2
   mtu 9214
   no switchport
   ip address 10.0.0.11/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.3
!
interface Loopback0
   ip address 10.1.1.13/32
   ip ospf area 0.0.0.3
!
ip routing
!
ip route 8.8.8.8/32 Null0
!
router ospf 100
   router-id 5.5.5.5
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   redistribute static
   area 0.0.0.3 nssa
   network 10.0.0.4/31 area 0.0.0.3
   network 10.0.0.10/31 area 0.0.0.3
   max-lsa 12000
!
```
