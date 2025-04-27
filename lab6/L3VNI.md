### Цель лабораторной работы:
1. Настроить хосты из разных подсетей в разных vlan за конкретными VTEP.
2. Настроить роутинг между хостами разных VTEP через L3VNI.
3. Проверить связность между хостами разных VTEP через ip-vrf.

Прикладываю схему лабораторной работы:

![image](https://github.com/user-attachments/assets/b612e266-6d68-41d6-bb72-6d6ec9258d6e)

# Типовая конфигурация leaf:

```bash
vlan 11
   name L3_Test
!
vrf instance L3_Test
!
interface Ethernet4
   description to_PC2
   switchport access vlan 11
!
interface Vlan11
   vrf L3_Test
   ip address 172.16.16.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 11 vni 1111
   vxlan vrf L3_Test vni 666
!
ip routing
ip routing vrf L3_Test
!
router bgp 65001
  !
   vlan 11
      rd 10.1.1.11:11
      route-target both 11:1111
      redistribute learned
   !
   vrf L3_Test
      rd 10.1.1.11:666
      route-target import evpn 666:1
      route-target export evpn 666:1
      redistribute connected
      redistribute static
!
```

# Разберем типовой конфиг Leaf:
1)

```bash
router bgp 65001 - Здесь описывается создание mac-vrf для этого vlan 11.
  !
   vlan 11
      rd 10.1.1.11:11
      route-target both 11:1111
      redistribute learned
```

2)

```bash
   vrf L3_Test - создание самого ip-vrf
      rd 10.1.1.11:666
      route-target import evpn 666:1
      route-target export evpn 666:1
      redistribute connected
      redistribute static
!
```
3)

```bash
vxlan vlan 11 vni 1111 - добавление mac-vrf/ip-vrf в vxlan
vxlan vrf L3_Test vni 666
```

# Теперь проверим ping между нашими компьютерами, живущими за vlan11 и vlan 12, то есть leaf-1/leaf-2:

```bash
VPCS> ping 172.16.16.2
84 bytes from 172.16.16.2 icmp_seq=1 ttl=62 time=896.419 ms
84 bytes from 172.16.16.2 icmp_seq=2 ttl=62 time=75.561 ms
84 bytes from 172.16.16.2 icmp_seq=3 ttl=62 time=75.285 ms
84 bytes from 172.16.16.2 icmp_seq=4 ttl=62 time=63.981 ms
84 bytes from 172.16.16.2 icmp_seq=5 ttl=62 time=59.519 ms
```
Связность имеется.

# Рассмотрим, как выглядят поля NLRI для mac-vrf и ip-vrf:

![image](https://github.com/user-attachments/assets/0d20169f-10db-4b2e-af64-61abca70e1c9)

Как видим, здесь мы поймали RD для mac-vrf, но в community целых 4 значения:
1.RT для mac-vrf 13:1313
2.RT для ip-vrf 666:1
3.Указание о том, что это Vxlan
4.Transitive EVPN - дамп был снят со Spine, это означает, что пусть даже Spine не понимает этого аттрибута, он должен быть сохранен и передан дальше на другие Leaf.

# Заключительные размышления:

1) Также на leaf-3 в этой лабораторной работе я не стал настраиваеть mac-vrf, так как за каждым Leaf у меня закреплен лишь один Vlan, лишь один хост. Поэтому трафик между L2-сегментами за одним Vtep или между двумя Vtep, между котороыми растянут L2, в моем случае не подразумевается. То есть трафик их PC за Vlan13 можем идти только к хостам во Vlan11-12, которые расположены за другими Vtep и могут быть известны нам через ip-vrf, поэтому в любом исходе пакеты "улетят" в ip-vrf.

2) Когда нам необходим и mac-vrf, и ip-vrf на одном Vtep?
A: Если в будущем во VLAN добавятся хосты, без mac-vrf даже сообщения между этими хостами будут сначала уходить в ip-vrf, а затем опускаться обратно на Vtep.
B: Если нужно растянуть VLAN между VTEP, тогда mac-vrf обязателен, чтобы организовать l2vni между Vtep.
