В процессе...
Схема данной лабораторной работы:
![image](https://github.com/user-attachments/assets/c197fef2-7609-4292-ba7a-1737a5871794)

```bash
Сразу небольшая ремарочка:
Столкнулся с проблемой, что виртуальная Ариста отнимает 3 байта от MTU на интерфейсе. У меня был указан везде MTU 9214, при этом вывод isis interface давал значение в 9211:
```bash
Spine-1#sh isis Test interface

IS-IS Instance: Test VRF: default

  Interface Ethernet1:
    Index: 14 SNPA: P2P
    MTU: 9211 Type: point-to-point
```
При этом MTU на обоих Spine/Leaf были одинаковые с точки зрения isis - 9211. Но соседство не хотело подниматься. 

Пришлось убрать MTU с интерфейсов, только тогда соседство поднялось:
### IS-IS Neighbors for Instance Test on Leaf-1

| Instance | VRF     | System Id | Type | Interface | SNPA | State | Hold Time | Circuit Id |
|----------|---------|-----------|------|-----------|------|-------|-----------|------------|
| Test     | default | Spine-1   | L1   | Ethernet1 | P2P  | UP    | 27        | 0E         |
```
