# Проектирование сетевой фабрики на основне VxLAN EVPN

### Цель проекта:

```bash
1. Спроектировать взаимодействие клиентов, чьи Дата-центры находятся в разных PODах, а также имеют физические стыки с офисом клиента вне облачного провайдера.
2. Описать предъявляемые требования и применяемые технологии для решения задачи.
3. Выполнить конфигурацию оборудования и продемонстрировать работоспособность услуги.
```

### Описание:
В этой работе я бы хотел воссоздать облако, состоящее из 4-ых площадок, которые могут работать как на раз платформах виртуализации, так и на одной единой. Самое главное, что гипервизоры подключаются в EVPN-фабрику, которая реализована в моей работе на Arista. 

Для упрощения понимания, один клиент хочет услугу L3VPN, которую я назвал так же для упрощения L3VNI. А также есть второй клиент в этой фабрике, которому достаочно L2VNI. 
Клиенту L3VNI необходим также выход из фабрики в свой офис, который находится на стыке с площадкой EXT. 

На стыке с клиентом поднимается отдельный eBGP-пиринг в его VRF, что позволяет получать от офиса и отдавать из EVPN-фабрики префиксы. 
Прикладываю ниже типовую схему взаимодействия. К сожалению, в лабораторном проекте такое множество устройств добавить и настроить возможности нет:

![image](https://github.com/user-attachments/assets/c7ad4f0a-1d78-4856-b873-cbb7ca0d6182)

### Требования к проекту:

```bash
1.Обеспечить сетевую связность между всеми точками клиента и отразить ее выводами команд.
2.Приложить конфигурацию всех устройств.
3.Обосновать выбор настраиваемых технологий.
```

### Адресный план:

```bash
POD	Host	        Interface	IP address/vlan	Description	
PD01	PD01-Spine-1	Loopback 0	1.1.1.1 	Lo0	
PD01	PD01-Spine-1	Eth1	        10.10.10.0	leaf-1	
PD01	PD01-Spine-1	Eth2	        10.10.10.2	leaf-2	
PD01	PD01-leaf-1	Loopback 0	1.1.1.2 	Lo0	
PD01	PD01-leaf-1	Eth1	        10.10.10.1	Spine	
PD01	PD01-leaf-1	Eth2	        Vlan111	        sw1	
PD01	PD01-leaf-1	Po1	        Trunk	        LACP	
PD01	PD01-leaf-2	Loopback 0	1.1.1.3 	Lo0	
PD01	PD01-leaf-2	Eth1	        10.10.10.3	Spine	
PD01	PD01-leaf-2	Eth2	        Vlan666	        sw1	
PD01	PD01-leaf-2	Po1	        Trunk	        LACP	
PD01	PD01-sw-1	Eth1	        Vlan111	        leaf-1	
PD01	PD01-sw-1	Eth2	        Vlan666	        leaf-2	
PD01	PD01-sw-1	Eth7	        192.168.1.0/24	srv-1	
PD01	PD01-sw-1	Eth8	        192.168.66.0/24	srv-2	
PD01	PD01-sw-1	Po1	        Trunk	        LACP	
					
PD02	PD02-Spine-1	Loopback 0	2.2.2.1	Lo0	
PD02	PD02-Spine-1	Eth1	        20.20.20.0	leaf-1	
PD02	PD02-Spine-1	Eth2	        20.20.20.2	leaf-2	
PD02	PD02-Spine-1	Eth3	        40.40.40.0	ext-leaf	
PD02	PD02-leaf-1	Loopback 0	2.2.2.2 	Lo0	
PD02	PD02-leaf-1	Eth1	        20.20.20.1	Spine	
PD02	PD02-leaf-1	Eth2	        Vlan222 	sw1	
PD02	PD02-leaf-1	Po1	        Trunk    	LACP	
PD02	PD02-leaf-2	Loopback 0	2.2.2.3 	Lo0	
PD02	PD02-leaf-2	Eth1	        20.20.20.3	Spine	
PD02	PD02-leaf-2	Eth2	        Vlan666	        sw1	
PD02	PD02-leaf-2	Po1	        Trunk	        LACP	
PD02	PD02-sw-1	Eth1	        Vlan222 	leaf-1	
PD02	PD02-sw-1	Eth2	        Vlan666	        leaf-2	
PD02	PD02-sw-1	Eth7	        192.168.2.0/24	srv-1	
PD02	PD02-sw-1	Eth8	        192.168.66.0/24	srv-2	
PD02	PD02-sw-1	Po1	        Trunk	        LACP	
					
					
PD03	PD03-Spine-1	Loopback 0	3.3.3.1	Lo0	
PD03	PD03-Spine-1	Eth1    	30.30.30.0	leaf-1	
PD03	PD03-Spine-1	Eth2    	30.30.30.2	leaf-2	
PD03	PD03-leaf-1	Loopback 0	3.3.3.2 	Lo0	
PD03	PD03-leaf-1	Eth1    	30.30.30.1	Spine	
PD03	PD03-leaf-1	Eth2    	Vlan333 	sw1	
PD03	PD03-leaf-1	Po1     	Trunk   	LACP	
PD03	PD03-leaf-2	Loopback 0	3.3.3.3 	Lo0	
PD03	PD03-leaf-2	Eth1    	30.30.30.3	Spine	
PD03	PD03-leaf-2	Eth2    	Vlan666 	sw1	
PD03	PD03-leaf-2	Po1     	Trunk   	LACP	
PD03	PD03-sw-1	Eth1    	Vlan333 	leaf-1	
PD03	PD03-sw-1	Eth2    	Vlan666 	leaf-2	
PD03	PD03-sw-1	Eth7    	192.168.3.0/24	srv-1	
PD03	PD03-sw-1	Eth8    	192.168.66.0/24	srv-2	
PD03	PD03-sw-1	Po1     	Trunk   	LACP	
					
EXT	EXT-leaf-1	Loopback 0	4.4.4.1 	Lo0	
EXT	EXT-leaf-1	Eth1    	40.40.40.1	Spine	
EXT	EXT-leaf-1	Eth2    	50.50.50.0	Client_Office	
EXT	EXT-leaf-1	Vlan444 	172.16.0.0/24	ext_vlan	
Client_Office	   	Loopback 0	5.5.5.1 	Lo0	
Client_Office		Eth1    	50.50.50.1	Ext-leaf	
Client_Office		Eth7    	Vlan555 	office_srv	
```

### Настройка POD:

Для обеспечения связности между Loopback-адресами я выбрал протокол eBGP, на физических стыках я поднимаю eBGP-пиринг, а затем анонсирую с них адреса Loopback.
Далее я поднимаю eBGP пиринг между Leaf-Spine по их Loopback 0. Ниже приведу типовые настройки Leaf и Spine:
PD01-leaf-1:

```bash
hostname PD01-leaf-1
!
spanning-tree mode mstp
!
vlan 111
   name for_L3VNI
!
vlan 666
   name for_L2VNI
!
vrf instance L3VNI
!
interface Port-Channel1
   mtu 9214
   switchport trunk allowed vlan 111,666
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:1111
      route-target import 00:00:00:00:00:11
   lacp system-id 0010.0101.0011
!
interface Ethernet1
   description up_pd01-spine-1
   no switchport
   ip address 10.10.10.1/31
!
interface Ethernet2
   description down_pd01-sw-1
   channel-group 1 mode active
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 1.1.1.2/32
!
interface Management1
!
interface Vlan111
   vrf L3VNI
   ip address 192.168.1.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 111 vni 10111
   vxlan vlan 666 vni 10666
   vxlan vrf L3VNI vni 10777
!
ip routing
ip routing vrf L3VNI
!
ip prefix-list PL_CONNECT seq 10 permit 1.1.1.2/32
!
route-map RM_CONNECT permit 10
   match ip address prefix-list PL_CONNECT
!
router bgp 65010
   router-id 1.1.1.2
   no bgp default ipv4-unicast
   maximum-paths 10 ecmp 10
   neighbor SPINE peer group
   neighbor SPINE send-community
   neighbor SPINE maximum-routes 100
   neighbor SPINE_EVPN peer group
   neighbor SPINE_EVPN update-source Loopback0
   neighbor SPINE_EVPN allowas-in 3
   neighbor SPINE_EVPN ebgp-multihop 5
   neighbor SPINE_EVPN send-community
   neighbor 1.1.1.1 peer group SPINE_EVPN
   neighbor 1.1.1.1 remote-as 65001
   neighbor 1.1.1.1 description pd01-Spine-1
   neighbor 1.1.1.1 allowas-in 3
   neighbor 10.10.10.0 peer group SPINE
   neighbor 10.10.10.0 remote-as 65001
   neighbor 10.10.10.0 description pd01-Spine-1
   neighbor 10.10.10.0 allowas-in 3
   no neighbor 10.10.10.1 allowas-in
   redistribute connected route-map RM_CONNECT
   !
   vlan 111
      rd 1.1.1.2:111
      route-target both 10:1110
      redistribute learned
   !
   vlan 666
      rd 1.1.1.2:666
      route-target both 10:6660
      redistribute learned
   !
   address-family evpn
      neighbor SPINE_EVPN activate
   !
   address-family ipv4
      neighbor SPINE activate
   !
   vrf L3VNI
      rd 1.1.1.2:777
      route-target import evpn 777:1
      route-target export evpn 777:1
      redistribute connected
      redistribute static
!
end
```
И настройка PD01-Spine-1:

```bash
PD01-Spine-1#sh run
! Command: show running-config
! device: PD01-Spine-1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
username bartnik role network-admin secret sha512 $6$fO2caICAo8lwS1QM$GaWxFbSSMHF19ooXA7EyJw2xwWzAVajCQnPa4yhdzuJKa.sw.8DhH2aIv5iEcFknBBG2PqnMWjYLenk4ZP1iW0
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname PD01-Spine-1
!
spanning-tree mode mstp
!
interface Ethernet1
   description down_pd01-leaf-1
   no switchport
   ip address 10.10.10.0/31
!
interface Ethernet2
   description down_pd01-leaf-2
   no switchport
   ip address 10.10.10.2/31
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
   description PD02-spine-1
   no switchport
   ip address 172.16.0.0/31
!
interface Loopback0
   ip address 1.1.1.1/32
!
interface Management1
!
ip routing
!
ip prefix-list PL_CONNECT seq 10 permit 1.1.1.1/32
!
route-map RM_CONNECT permit 10
   match ip address prefix-list PL_CONNECT
!
router bgp 65001
   router-id 1.1.1.1
   no bgp default ipv4-unicast
   neighbor BORDER peer group
   neighbor BORDER send-community
   neighbor BORDER maximum-routes 100
   neighbor LEAF peer group
   neighbor LEAF send-community
   neighbor LEAF maximum-routes 100
   neighbor LEAF_EVPN peer group
   neighbor LEAF_EVPN next-hop-unchanged
   neighbor LEAF_EVPN update-source Loopback0
   neighbor LEAF_EVPN ebgp-multihop 5
   neighbor LEAF_EVPN send-community extended
   neighbor 1.1.1.2 peer group LEAF_EVPN
   neighbor 1.1.1.2 remote-as 65010
   neighbor 1.1.1.2 description pd01-leaf-1
   neighbor 1.1.1.3 peer group LEAF_EVPN
   neighbor 1.1.1.3 remote-as 65010
   neighbor 1.1.1.3 description pd01-leaf-2
   neighbor 10.10.10.1 peer group LEAF
   neighbor 10.10.10.1 remote-as 65010
   neighbor 10.10.10.1 description pd01-leaf-1
   neighbor 10.10.10.3 peer group LEAF
   neighbor 10.10.10.3 remote-as 65010
   neighbor 10.10.10.3 description pd01-leaf-2
   neighbor 172.16.0.1 peer group BORDER
   neighbor 172.16.0.1 remote-as 65002
   neighbor 172.16.0.1 description PD02-spine-1
   redistribute connected route-map RM_CONNECT
   !
   address-family evpn
      neighbor BORDER activate
      neighbor LEAF_EVPN activate
   !
   address-family ipv4
      neighbor BORDER activate
      neighbor LEAF activate
!
end
```
К Leaf-коммутаторам ко всем PD подключены PD-sw-1, они служат коммутаторами, в которые включаются клиентские хосты, по задумке, на этих SW есть стык с физическими серверами клиента, на которых крутится какая-либо виртуализация. К гипервизорам через bridge-ноды подключены SW во всех PD, трафик инкапсулируется во Vlan и проходит в Trunk-порт в сторону Leaf. Порты в сторону Leaf собраны в Po1. Leaf-коммутаторы в свою очеред собраны в evpn ESI, чтобы исключить петли на этом уровне и обеспечить отказоустойчивость.

Настройка SW довольно проста:

```bash
PD01-sw-1# sh run
! Command: show running-config
! device: PD01-sw-1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
username bartnik role network-admin secret sha512 $6$0Cbx.6i.DVZeu/mU$F2knRUthsL                                                                                                                                                             1qKtQm5yWFffv/qv7GOkhABtrZ5CMxtY68KE6pUI6/znc1iShqnBJTCfXIu4I8J6ykgFsgvixlj.
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname PD01-sw-1
!
spanning-tree mode mstp
!
vlan 111
   name for_L3VNI
!
vlan 666
   name for_L2VNI
!
interface Port-Channel1
   description up_pd01-leaf-1|leaf-2
   mtu 9214
   switchport mode trunk
   lacp system-id 0010.0101.0001
!
interface Ethernet1
   description pd01-leaf-1
   channel-group 1 mode active
!
interface Ethernet2
   description pd01-leaf-2
   channel-group 1 mode active
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
   description pd01-srv-1
   switchport access vlan 111
!
interface Ethernet8
   description pd01-srv-2
   switchport access vlan 666
!
interface Management1
!
no ip routing
!
end
```
