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
Leaf_3 перенесен в NSSA-зону, а также имеет мнимый маршрут до 8.8.8.8 через Null0, интерфейс выбрна для наглядности распространения LSA.

Хотелось бы отметить также момент подвисания соседства  exchange, если mtu на интерфейсах не совпадает, убрал mtu 9214 с Leaf_2 в сторону Spine_1:
![image](https://github.com/user-attachments/assets/22136e5f-6021-417b-850b-7ec7db0800b2)

Затем возвращаем MTU на Leaf_2 eth1 обратно в исходное значение 9214:
![image](https://github.com/user-attachments/assets/aa903000-9ace-4b0b-ba1b-da434f4f714b)

После конфигурации всех устройств проверим Database на Spine_2:
### OSPF Router Information

**Router with ID (2.2.2.2) (Instance ID 100) (VRF default)**

---

### Router Link States (Area 0.0.0.0)

| Link ID | ADV Router | Age | Seq# | Checksum | Link count |
|---------|------------|-----|------|----------|------------|
| 1.1.1.1 | 1.1.1.1    | 259 | 0x8000000b | 0x367f   | 3          |
| 2.2.2.2 | 2.2.2.2    | 27  | 0x8000000e | 0xbae2   | 3          |
| 3.3.3.3 | 3.3.3.3    | 454 | 0x80000007 | 0x3c13   | 5          |

---

### Summary Link States (Area 0.0.0.0)

| Link ID  | ADV Router | Age  | Seq# | Checksum |
|----------|------------|------|------|----------|
| 10.1.1.12| 1.1.1.1    | 19   | 0x80000003 | 0x1eec   |
| 10.0.0.10| 2.2.2.2    | 524  | 0x80000002 | 0xc254   |
| 10.0.0.4 | 1.1.1.1    | 1219 | 0x80000003 | 0x1b05   |
| 10.0.0.8 | 1.1.1.1    | 19   | 0x80000003 | 0x57ba   |
| 10.0.0.8 | 2.2.2.2    | 524  | 0x80000002 | 0xd642   |
| 10.0.0.4 | 2.2.2.2    | 404  | 0x80000002 | 0x63af   |
| 10.0.0.2 | 1.1.1.1    | 1519 | 0x80000003 | 0x2ff2   |
| 10.0.0.2 | 2.2.2.2    | 464  | 0x80000002 | 0x779d   |
| 10.0.0.10| 1.1.1.1    | 1219 | 0x80000003 | 0x43cc   |
| 10.1.1.13| 2.2.2.2    | 45   | 0x80000001 | 0xf90e   |
| 10.1.1.12| 2.2.2.2    | 464  | 0x80000002 | 0x206    |
| 10.1.1.13| 1.1.1.1    | 47   | 0x80000001 | 0x18f3   |

---

### ASBR Summary Link States (Area 0.0.0.0)

| Link ID | ADV Router | Age | Seq# | Checksum |
|---------|------------|-----|------|----------|
| 2.2.2.2 | 1.1.1.1    | 439 | 0x80000002 | 0xc753   |
| 1.1.1.1 | 2.2.2.2    | 464 | 0x80000002 | 0xd743   |

---

### Router Link States (Area 0.0.0.1)

| Link ID | ADV Router | Age | Seq# | Checksum | Link count |
|---------|------------|-----|------|----------|------------|
| 1.1.1.1 | 1.1.1.1    | 19  | 0x80000006 | 0xb227   | 2          |
| 4.4.4.4 | 4.4.4.4    | 438 | 0x80000010 | 0x39fb   | 5          |
| 2.2.2.2 | 2.2.2.2    | 464 | 0x80000007 | 0x3d87   | 2          |

---

### Summary Link States (Area 0.0.0.1)

| Link ID  | ADV Router | Age  | Seq# | Checksum |
|----------|------------|------|------|----------|
| 10.1.1.11| 1.1.1.1    | 259  | 0x80000004 | 0x26e4   |
| 10.0.0.0 | 1.1.1.1    | 1519 | 0x80000003 | 0x43e0   |
| 10.0.0.4 | 1.1.1.1    | 1219 | 0x80000003 | 0x1b05   |
| 10.0.0.0 | 2.2.2.2    | 464  | 0x80000002 | 0x8b8b   |
| 10.0.0.4 | 2.2.2.2    | 404  | 0x80000003 | 0x61b0   |
| 10.0.0.10| 2.2.2.2    | 524  | 0x80000002 | 0xc254   |
| 10.0.0.10| 1.1.1.1    | 1219 | 0x80000003 | 0x43cc   |
| 10.0.0.6 | 1.1.1.1    | 259  | 0x80000004 | 0x69a9   |
| 10.0.0.6 | 2.2.2.2    | 524  | 0x80000002 | 0xea30   |
| 10.1.1.2 | 2.2.2.2    | 27   | 0x80000001 | 0x419    |
| 10.1.1.2 | 1.1.1.1    | 29   | 0x80000001 | 0xea22   |
| 10.1.1.1 | 1.1.1.1    | 1519 | 0x80000003 | 0x28f7   |
| 10.1.1.13| 2.2.2.2    | 45   | 0x80000001 | 0xf90e   |
| 10.1.1.1 | 2.2.2.2    | 464  | 0x80000002 | 0xd434   |
| 10.1.1.13| 1.1.1.1    | 47   | 0x80000001 | 0x18f3   |
| 10.1.1.11| 2.2.2.2    | 464  | 0x80000002 | 0xcfc    |

---

### Router Link States (Area 0.0.0.3)

| Link ID | ADV Router | Age | Seq# | Checksum | Link count |
|---------|------------|-----|------|----------|------------|
| 5.5.5.5 | 5.5.5.5    | 46  | 0x80000010 | 0x9388   | 5          |
| 2.2.2.2 | 2.2.2.2    | 464 | 0x80000009 | 0x496b   | 2          |
| 1.1.1.1 | 1.1.1.1    | 1219| 0x8000000a | 0xba0d   | 2          |

---

### Summary Link States (Area 0.0.0.3)

| Link ID  | ADV Router | Age  | Seq# | Checksum |
|----------|------------|------|------|----------|
| 10.1.1.12| 1.1.1.1    | 19   | 0x80000003 | 0xc341   |
| 10.0.0.8 | 2.2.2.2    | 524  | 0x80000002 | 0x7c96   |
| 10.0.0.0 | 1.1.1.1    | 1039 | 0x80000004 | 0xe636   |
| 10.0.0.0 | 2.2.2.2    | 464  | 0x80000002 | 0x31df   |
| 10.0.0.8 | 1.1.1.1    | 19   | 0x80000003 | 0xfc0f   |
| 10.0.0.2 | 1.1.1.1    | 1039 | 0x80000004 | 0xd248   |
| 10.0.0.2 | 2.2.2.2    | 464  | 0x80000003 | 0x1bf2   |
| 10.0.0.6 | 1.1.1.1    | 259  | 0x80000005 | 0xdfe    |
| 10.0.0.6 | 2.2.2.2    | 524  | 0x80000002 | 0x9084   |
| 10.1.1.1 | 1.1.1.1    | 1039 | 0x80000004 | 0xcb4d   |
| 10.1.1.2 | 2.2.2.2    | 27   | 0x80000001 | 0xa96d   |
| 10.1.1.12| 2.2.2.2    | 464  | 0x80000003 | 0xa55b   |
| 10.1.1.2 | 1.1.1.1    | 29   | 0x80000001 | 0x9076   |
| 10.1.1.11| 1.1.1.1    | 259  | 0x80000005 | 0xc93a   |
| 10.1.1.1 | 2.2.2.2    | 464  | 0x80000002 | 0x7a88   |
| 10.1.1.11| 2.2.2.2    | 464  | 0x80000002 | 0xb151   |

---

### NSSA Link States (Area 0.0.0.3)

| Link ID | ADV Router | Age | Seq# | Checksum |
|---------|------------|-----|------|----------|
| 8.8.8.8 | 5.5.5.5    | 46  | 0x80000003 | 0xa8af   |

---

### Type-5 AS External Link States

| Link ID | ADV Router | Age | Seq# | Checksum | Tag |
|---------|------------|-----|------|----------|-----|
| 8.8.8.8 | 2.2.2.2    | 40  | 0x80000003 | 0x97d6   | 0   |

Как видим из Database, LSA 7 от Leaf_3 прилетел на Spine_2/Spine_1, затем Spine_2/Spine_1 отдают эту информацию как LSA 5 для соседней Area 1.
Вывод с Leaf-2:
### Type-5 AS External Link States

| Link ID | ADV Router | Age  | Seq# | Checksum | Tag |
|---------|------------|------|------|----------|-----|
| 8.8.8.8 | 2.2.2.2    | 1299 | 0x80000003 | 0x97d6 | 0   |



