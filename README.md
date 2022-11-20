
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
В выводе команды zfs get all | grep compression видно, что на каждую файловую систему назначен свой алгоритм сжатия:
```
otus1  compression           lzjb                   local
otus2  compression           lz4                    local
otus3  compression           gzip-9                 local
otus4  compression           zle                    local
```
5. Скачиваем один и тот же текстовый файл во все пулы:
```
for i in {1..4}; do wget -P /otus$i http://www.gutenberg.org/ebooks/2600.txt.utf-8; done
```
6. Проверяем сколько места занимает этот файл в фалойвых системах с разным алгоритмом сжатия с помощью команды zfs list:
```
NAME    USED  AVAIL     REFER  MOUNTPOINT
otus1  2.48M   350M     2.41M  /otus1
otus2  2.09M   350M     2.02M  /otus2
otus3  1.30M   351M     1.23M  /otus3
otus4  3.30M   349M     3.23M  /otus4
```
Команда zfs get all | grep compressratio | grep -v ref покажет степень сжатия файлов:
```
otus1  compressratio         1.35x                  -
otus2  compressratio         1.62x                  -
otus3  compressratio         2.64x                  -
otus4  compressratio         1.01x                  -
```
Вывод. Таким образом, мы увидели, что наилучшей степенью сжатия обладает алгоритм gzip-9 : 2.64x
