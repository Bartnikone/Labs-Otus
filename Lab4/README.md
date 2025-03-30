Цель лабораторной работы:
1. Настроить протокол eBGP.
2. Ограничить передаваемую информацию между Leaf до адресов Loopback.
3. Настроить peer group для Spine/Leaf для более удобного управления соседями.
4. Включить ECMP и проверить, что адреса Loopback Leaf доступны сразу через два Spine.

Итак, схема пока что остается неизменной:

![image](https://github.com/user-attachments/assets/31267cb2-e537-48a4-9f96-07ad4c84fce0)

Рассмотрим конфигурацию Spine на примере Spine-1:

```bash

ip prefix-list PL_LO_LEAFS seq 10 permit 10.1.1.8/29 le 32 - создадим PL для разрешения лишь адресов Lo0 от наших трех Leaf
!
route-map RM_LEAFS permit 10 - создадим RM и свяжем ее с PL
   match ip address prefix-list PL_LO_LEAFS
!
router bgp 65000
   router-id 10.1.1.1
   no bgp default ipv4-unicast - отключаем автоматический переход к IPv4
   maximum-paths 10 ecmp 10 - включаем ECMP
   neighbor LEAFS peer group - создаем группу соседей с именем LEAFS
   neighbor LEAFS bfd - включаем BFD для всех наших соседей
   neighbor LEAFS timers 3 9 - Keepalive 3, Hold 9
   neighbor LEAFS route-map RM_LEAFS in - принимаем лишь адреса Lo0
   neighbor LEAFS maximum-routes 1000 - лимит маршрутов(защита в случае DDos)
   neighbor 10.0.0.1 peer group LEAFS
   neighbor 10.0.0.1 remote-as 65001
   neighbor 10.0.0.3 peer group LEAFS
   neighbor 10.0.0.3 remote-as 65002
   neighbor 10.0.0.5 peer group LEAFS
   neighbor 10.0.0.5 remote-as 65003
   !
   address-family ipv4
      neighbor LEAFS activate - и активируем соседа в AF ipv4
```
И конфигуарция для Leaf-1:

```bash

ip prefix-list PL_CONNECT seq 10 permit 10.1.1.8/29 le 32
!
route-map RM_REDIS_CON permit 10
   match ip address prefix-list PL_CONNECT
!
router bgp 65001
   router-id 10.1.1.11
   no bgp default ipv4-unicast
   maximum-paths 10 ecmp 10
   neighbor SPINES peer group
   neighbor SPINES remote-as 65000
   neighbor SPINES bfd
   neighbor SPINES timers 3 9
   neighbor SPINES maximum-routes 1000
   neighbor 10.0.0.0 peer group SPINES
   neighbor 10.0.0.6 peer group SPINES
   redistribute connected route-map RM_REDIS_CON
   !
   address-family ipv4
      neighbor SPINES activate
!
```

Команда "no bgp default ipv4-unicast" в глобальной конфигурации BGP нужна для того, чтобы исключить автоматическую активацию IPv4 для всех соседей. Соседи не будут пытаться активировать сессии Ipv4, если мы явно не активируем этих соседей в AF IPv4. Зачем это нужно? Чтобы четко контролировать, какие AF нам нужны для работы underlay и overlay.

Для включения ECMP прописываем в BGP :"maximum-paths 10 ecmp 10" - команда "maximum-paths 10" позволяет сохранить в нашем случае 10 путей в таблице BGP для какого-либо префикса, полученного от Leaf. А команда "ecmp 10" позволяет добавить их в FIB.

neighbor SPINES timers 3 9 - меняем таймеры со стандартных 60/180 на 3 секунды Keepalive и 9 секунды Hold. Более агрессивные таймеры для скорости реакции на сессиях.

И задаем максимальное количество маршрутов для принятия до 1000 - neighbor SPINES maximum-routes 1000. В этой лабораторной, конечно, число слишком высокое, можно было бы на данном этапе уменьшить до 10. Spine принимают по 3 префикса /32(адреса Lo0 на Leaf), а Leaf принимают по 2 префикса /32 от соседних Leaf. Число в 1000 - "запас прочности" с заделом на будущее).

Посмотрим на таблицу маршрутов на Leaf-1:

Gateway of last resort is not set

```bash
 C        10.0.0.0/31 is directly connected, Ethernet1
 C        10.0.0.6/31 is directly connected, Ethernet2
 C        10.1.1.11/32 is directly connected, Loopback0
 B E      10.1.1.12/32 [200/0] via 10.0.0.0, Ethernet1
                               via 10.0.0.6, Ethernet2
 B E      10.1.1.13/32 [200/0] via 10.0.0.0, Ethernet1
                               via 10.0.0.6, Ethernet2
```

Как видно из таблицы, нам доступны оба соседних Lo0 от Leaf-2/3 сразу через двух Spine. ECMP работает.

