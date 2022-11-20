
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

II. Приступаем ко второй части задания: импорт пула, и определение его настроек.
1. Скачиваем архив с пулом:
```
wget -O archive.tar.gz --no-check-certificate 'https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download'
```
Разархивируем его с помощью команды tar -xzvf archive.tar.gz :
```
zpoolexport/
zpoolexport/filea
zpoolexport/fileb
```
2. Проверяем, можно ли произвести импорт:
```
[root@zfs ~]# zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

	otus                         ONLINE
	  mirror-0                   ONLINE
	    /root/zpoolexport/filea  ONLINE
	    /root/zpoolexport/fileb  ONLINE
```
3. Делаем импорт пула и проверяем статус пулов:
```
[root@zfs ~]# zpool import -d zpoolexport/ otus
[root@zfs ~]# zpool status
  pool: otus
 state: ONLINE
  scan: none requested
config:

	NAME                         STATE     READ WRITE CKSUM
	otus                         ONLINE       0     0     0
	  mirror-0                   ONLINE       0     0     0
	    /root/zpoolexport/filea  ONLINE       0     0     0
	    /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors

  pool: otus1
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus1       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus2       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdd     ONLINE       0     0     0
	    sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus3       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdf     ONLINE       0     0     0
	    sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus4       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdh     ONLINE       0     0     0
	    sdi     ONLINE       0     0     0

errors: No known data errors
```
4. Делаем запрос параметров пула:
```
[root@zfs ~]# zpool get all otus
NAME  PROPERTY                       VALUE                          SOURCE
otus  size                           480M                           -
otus  capacity                       0%                             -
otus  altroot                        -                              default
otus  health                         ONLINE                         -
otus  guid                           6554193320433390805            -
otus  version                        -                              default
otus  bootfs                         -                              default
otus  delegation                     on                             default
otus  autoreplace                    off                            default
otus  cachefile                      -                              default
otus  failmode                       wait                           default
otus  listsnapshots                  off                            default
otus  autoexpand                     off                            default
otus  dedupditto                     0                              default
otus  dedupratio                     1.00x                          -
otus  free                           478M                           -
otus  allocated                      2.09M                          -
otus  readonly                       off                            -
otus  ashift                         0                              default
otus  comment                        -                              default
otus  expandsize                     -                              -
otus  freeing                        0                              -
otus  fragmentation                  0%                             -
otus  leaked                         0                              -
otus  multihost                      off                            default
otus  checkpoint                     -                              -
otus  load_guid                      4207918160654248604            -
otus  autotrim                       off                            default
otus  feature@async_destroy          enabled                        local
otus  feature@empty_bpobj            active                         local
otus  feature@lz4_compress           active                         local
otus  feature@multi_vdev_crash_dump  enabled                        local
otus  feature@spacemap_histogram     active                         local
otus  feature@enabled_txg            active                         local
otus  feature@hole_birth             active                         local
otus  feature@extensible_dataset     active                         local
otus  feature@embedded_data          active                         local
otus  feature@bookmarks              enabled                        local
otus  feature@filesystem_limits      enabled                        local
otus  feature@large_blocks           enabled                        local
otus  feature@large_dnode            enabled                        local
otus  feature@sha512                 enabled                        local
otus  feature@skein                  enabled                        local
otus  feature@edonr                  enabled                        local
otus  feature@userobj_accounting     active                         local
otus  feature@encryption             enabled                        local
otus  feature@project_quota          active                         local
otus  feature@device_removal         enabled                        local
otus  feature@obsolete_counts        enabled                        local
otus  feature@zpool_checkpoint       enabled                        local
otus  feature@spacemap_v2            active                         local
otus  feature@allocation_classes     enabled                        local
otus  feature@resilver_defer         enabled                        local
otus  feature@bookmark_v2            enabled                        local
```
5. Выводим информацию по отдельным параметрам пула:

  а. Размер пула:
```
[root@zfs ~]# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
```
  б. Тип пула:
```
[root@zfs ~]# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default
```
  в. Значение recordsize:
```
[root@zfs ~]# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local
```
  г. Какое сжатие используется:
```
[root@zfs ~]# zfs get compression otus
NAME  PROPERTY     VALUE     SOURCE
otus  compression  zle       local
```
  д. Тип контрольной суммы:
```
[root@zfs ~]# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
```
