
1.  С помощью команды lsblk смотрим список всех дисков, что есть в виртуальной машине:
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk 
`-sda1   8:1    0   40G  0 part /
sdb      8:16   0  512M  0 disk 
sdc      8:32   0  512M  0 disk 
sdd      8:48   0  512M  0 disk 
sde      8:64   0  512M  0 disk 
sdf      8:80   0  512M  0 disk 
sdg      8:96   0  512M  0 disk 
sdh      8:112  0  512M  0 disk 
sdi      8:128  0  512M  0 disk 
```
2. Создаём четыре пула по два диска в каждом в режиме RAID-1 (mirror):
```
zpool create otus1 mirror /dev/sdb /dev/sdc

zpool create otus2 mirror /dev/sdd /dev/sde

zpool create otus3 mirror /dev/sdf /dev/sdg

zpool create otus4 mirror /dev/sdh /dev/sdi
```
3. Выводим информацию о созданных пулах с помощью команды zpool list:
```
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus1   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus2   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus3   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus4   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
```
4. Добавим в каждую файловую систему алгоритм сжатия:
```
zfs set compression=lzjb otus1
zfs set compression=lz4 otus2
zfs set compression=gzip-9 otus3
zfs set compression=zle otus4
```
