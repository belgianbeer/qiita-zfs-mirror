# はじめに
筆者自宅のFreeBSDサーバーには、ストレージとしてSSD(256GB)が1台、HDD(2TB)が2台接続してあり全てZFSで構成して主にファイルサーバーとして利用しています。これらはSSDをOS用、HDDをデータ用として割り当てていますが、HDDが2台あるのはZFSでミラーとしているため容量としては1台分となります。

HDD側の使用量がそろそろ80%を超えてきているのと使用開始から10年以上経過していることがあり、さすがにそろそろ交換した方が良いという判断で6TBのHDDを2台用意して交換しました。この記事は実際にHDDを交換したときの記録です。

なおZFSは、空き容量が少なくなると他の多くのファイルシステムより極端に速度が低下する傾向があるため、容量ギリギリまで利用するのは得策ではありません。

# 事前の確認

HDDの交換作業に入るまえに、交換前の状態を確認して記録します。`zpool list`を実行すると次のように表示され、zrootがシステム用のSSD、zvol0がデータ用でHDDのミラー構成です。

```
$ zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
zroot   228G  16.8G   211G        -         -    12%     7%  1.00x  ONLINE  -
zvol0  1.81T  1.49T   333G        -         -     5%    82%  1.00x  ONLINE  -
$
```

ミラー構成の詳細は、`zpool list -v`か、`zpool status`で確認できます。

```
$ zpool list -v zvol0
NAME             SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  zvol0           1.81T  1.49T   333G        -         -     5%    82%  1.00x    ONLINE  -
  mirror-0      1.81T  1.49T   333G        -         -     5%  82.1%      -    ONLINE
    gpt/ndisk2      -      -      -        -         -      -      -      -    ONLINE
    gpt/ndisk1      -      -      -        -         -      -      -      -    ONLINE
$
$ zpool status zvol0
  pool: zvol0
 state: ONLINE
  scan: scrub repaired 0B in 05:36:58 with 0 errors on Tue Jan 17 08:40:57 2023
config:

        NAME            STATE     READ WRITE CKSUM
        zvol0           ONLINE       0     0     0
          mirror-0      ONLINE       0     0     0
            gpt/ndisk2  ONLINE       0     0     0
            gpt/ndisk1  ONLINE       0     0     0

errors: No known data errors
$
```

これらからディスクのndisk1とndisk2のパーティションでミラーを構成してあることが分かります。3台のディスクはada0(SSD)、ada1(HDD)、ada2(HDD)という構成なので、HDDのパーティション構成を確認します。

```
$ gpart show -l ada1 ada2
=>        40  3907029088  ada1  GPT  (1.8T)
          40  3907029088     1  ndisk1  (1.8T)

=>        40  3907029088  ada2  GPT  (1.8T)
          40  3907029088     1  ndisk2  (1.8T)

$
```
ada2、ada3はパーティションを作成するときにGPTで初期化して、ada1にはndisk1、ada2にはndisk2と名前をつけてありました。

# ミラー構成でのディスク交換の手順

ミラー(RAID 1)構成のHDDを2台とも入れ替える手順としては、ざっくり次の2種類の手順が考えられます。

* 新しいHDDでミラーを用意して既存のミラーからデータコピー後に入れ替える(あるいは先に交換してから元のディスクからデータを戻す)
* 片方のHDDを交換して一旦ミラーの機能でデータを同期して、完了後にもう一方のHDDを交換して改めてデータ同期する

前者の方法であればデータコピーは1回で済みますが同時に4台のHDDを接続する必要があります。この場合HDDはUSBで接続しても構わないですが、USB 2.0の転送速度では時間がかかりすぎるため最低でもUSB 3.0にしたいところです。実際このサーバーにはUSBは2.0であるため、筆者の場合は後者の1台づつ交換する方法を取りました。

# 1台目のHDDの交換

まず1台目のHDDの交換となりますが、ndisk2のパーティションのあるHDDを外して交換します。この場合サーバーをシャットダウンして電源を切り、古いHDDを撤去して新しいHDD接続後に電源を入れます。

## HDD交換時の注意

HDDを交換する場合に注意することは、電源を切る前にZFSプールの操作をおこなって、例えば `zpool detach zvol0 gpt/ndisk2` などを **実行してはいけません**。なぜならdetachでndisk2をプールから外した場合、ndisk2内のデータを消去したことになり、万が一新しいディスクとのデータの同期中にndisk1にエラーが発生した場合にndisk2に交換して同期をやり直すことができなくなってしまいます。電源を切ってndisk2を外すだけであればndisk2内のデータはndisk1と同じままになるので、いざというときにndisk2を使ってやり直しができます。

ディスクを交換して起動後に zvol0 の状態を確認します。
```
$ zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot   228G  16.8G   211G        -         -    12%     7%  1.00x    ONLINE  -
zvol0  1.81T  1.49T   333G        -         -     5%    82%  1.00x  DEGRADED  -
$
```
```
$ zpool status zvol0
  pool: zvol0
 state: DEGRADED
status: One or more devices could not be opened.  Sufficient replicas exist for
        the pool to continue functioning in a degraded state.
action: Attach the missing device and online it using 'zpool online'.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-2Q
  scan: scrub repaired 0B in 05:36:58 with 0 errors on Tue Jan 17 08:40:57 2023
config:

        NAME                      STATE     READ WRITE CKSUM
        zvol0                     DEGRADED     0     0     0
          mirror-0                DEGRADED     0     0     0
            12897545936258916783  UNAVAIL      0     0     0  was /dev/gpt/ndisk2
            gpt/ndisk1            ONLINE       0     0     0

errors: No known data errors
```
2台のHDDでミラーのはずが片側のgpt/ndisk2が無く1台だけになっているため、HEALTHやSTATEで「DEGRADED」と表示されています。

# ミラー同期前の準備 HDDの初期化

ミラーの同期を行うために、新しいディスクをGPTで初期化して、ZFS用のパーティションを用意します。新しいパーティションには古いHDDと区別するためsdisk1と名前をつけました。

まず、GPTでディスクを初期化します。
```
$ gpart show ada1
gpart: No such geom: ada1.
$ gpart create -s gpt ada1
ada1 created
$ gpart show ada1
=>         40  11721045088  ada1  GPT  (5.5T)
           40  11721045088        - free -  (5.5T)

$
```

続いてZFS用のパーティションを割り当てます。

```
$ gpart add -t freebsd-zfs -a 4k -l sdisk1 ada1
ada1p1 added
$ gpart show ada1
=>         40  11721045088  ada1  GPT  (5.5T)
           40  11721045088     1  freebsd-zfs  (5.5T)

$ gpart show -l ada1
=>         40  11721045088  ada1  GPT  (5.5T)
           40  11721045088     1  sdisk1  (5.5T)

$
```

パーティションを用意できたので、旧ndisk2をsdisk1で置き換えます。このための操作は`zpool replace`を使います。

```
$ zpool replace zvol0 12897545936258916783 gpt/sdisk1
$
```
これでsdisk1に対してndisk1からのミラーの同期が始まります。

同期が始まった様子を`zpool status`で確認します。
```
$ zpool status zvol0
  pool: zvol0
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Sat Jan 28 14:25:44 2023
        12.8G scanned at 596M/s, 600K issued at 27.3K/s, 1.49T total
        0B resilvered, 0.00% done, no estimated completion time
config:

        NAME                        STATE     READ WRITE CKSUM
        zvol0                       DEGRADED     0     0     0
          mirror-0                  DEGRADED     0     0     0
            replacing-0             DEGRADED     0     0     0
              12897545936258916783  UNAVAIL      0     0     0  was /dev/gpt/ndisk2
              gpt/sdisk1            ONLINE       0     0     0
            gpt/ndisk1              ONLINE       0     0     0

errors: No known data errors
$
```
# ★ここまで書いた
開始直後は終了予想時間は表示されません。
```
$ zpool status zvol0
  pool: zvol0
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Sat Jan 28 14:25:44 2023
        265G scanned at 664M/s, 6.51G issued at 16.3M/s, 1.49T total
        6.51G resilvered, 0.43% done, 1 days 02:27:43 to go
config:

        NAME                        STATE     READ WRITE CKSUM
        zvol0                       DEGRADED     0     0     0
          mirror-0                  DEGRADED     0     0     0
            replacing-0             DEGRADED     0     0     0
              12897545936258916783  UNAVAIL      0     0     0  was /dev/gpt/ndisk2
              gpt/sdisk1            ONLINE       0     0     0  (resilvering)
            gpt/ndisk1              ONLINE       0     0     0

errors: No known data errors
$ zpool status zvol0
```

```
$ zpool status zvol0
  pool: zvol0
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Sat Jan 28 14:25:44 2023
        265G scanned at 664M/s, 6.51G issued at 16.3M/s, 1.49T total
        6.51G resilvered, 0.43% done, 1 days 02:27:43 to go
config:

        NAME                        STATE     READ WRITE CKSUM
        zvol0                       DEGRADED     0     0     0
          mirror-0                  DEGRADED     0     0     0
            replacing-0             DEGRADED     0     0     0
              12897545936258916783  UNAVAIL      0     0     0  was /dev/gpt/ndisk2
              gpt/sdisk1            ONLINE       0     0     0  (resilvering)
            gpt/ndisk1              ONLINE       0     0     0

errors: No known data errors
$
```


```
$ zpool status zvol0
  pool: zvol0
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Sat Jan 28 14:25:44 2023
        289G scanned at 488M/s, 18.6G issued at 31.4M/s, 1.49T total
        18.6G resilvered, 1.22% done, 13:38:30 to go
config:

        NAME                        STATE     READ WRITE CKSUM
        zvol0                       DEGRADED     0     0     0
          mirror-0                  DEGRADED     0     0     0
            replacing-0             DEGRADED     0     0     0
              12897545936258916783  UNAVAIL      0     0     0  was /dev/gpt/ndisk2
              gpt/sdisk1            ONLINE       0     0     0  (resilvering)
            gpt/ndisk1              ONLINE       0     0     0

errors: No known data errors
$
```


```
minmin@brunehaut(2)$ zpool iostat  -v zvol0 1
                              capacity     operations     bandwidth
pool                        alloc   free   read  write   read  write
--------------------------  -----  -----  -----  -----  -----  -----
zvol0                       1.49T   333G     72     76  19.4M  19.1M
  mirror-0                  1.49T   333G     72     76  19.4M  19.1M
    replacing-0                 -      -      0    139     34  38.1M
      12897545936258916783      -      -      0      0      0      0
      gpt/sdisk1                -      -      0    139     34  38.1M
    gpt/ndisk1                  -      -     72      6  19.4M  60.3K
--------------------------  -----  -----  -----  -----  -----  -----
                              capacity     operations     bandwidth
pool                        alloc   free   read  write   read  write
--------------------------  -----  -----  -----  -----  -----  -----
zvol0                       1.49T   333G     87    261  87.0M  87.0M
  mirror-0                  1.49T   333G     88    261  88.0M  87.0M
    replacing-0                 -      -      0    261      0  87.0M
      12897545936258916783      -      -      0      0      0      0
      gpt/sdisk1                -      -      0    261      0  87.0M
    gpt/ndisk1                  -      -     87      0  88.0M      0
--------------------------  -----  -----  -----  -----  -----  -----
^C
minmin@brunehaut(2)$
                              capacity     operations     bandwidth
pool                        alloc   free   read  write   read  write
--------------------------  -----  -----  -----  -----  -----  -----
zvol0                       1.49T   333G     94    283  94.4M  94.4M
  mirror-0                  1.49T   333G     94    283  94.4M  94.4M
    replacing-0                 -      -      0    283      0  94.4M
      12897545936258916783      -      -      0      0      0      0
      gpt/sdisk1                -      -      0    283      0  94.4M
    gpt/ndisk1                  -      -     94      0  94.4M      0
--------------------------  -----  -----  -----  -----  -----  -----
                              capacity     operations     bandwidth
pool                        alloc   free   read  write   read  write
--------------------------  -----  -----  -----  -----  -----  -----
zvol0                       1.49T   333G     94    282  94.0M  93.0M
  mirror-0                  1.49T   333G     94    282  94.0M  93.0M
    replacing-0                 -      -      0    282      0  93.0M
      12897545936258916783      -      -      0      0      0      0
      gpt/sdisk1                -      -      0    282      0  93.0M
    gpt/ndisk1                  -      -     94      0  94.0M      0
--------------------------  -----  -----  -----  -----  -----  -----
^C
minmin@brunehaut(2)$
```

1時間後に確認したら残り6時間程度まで減っていました
```
$ zpool status zvol0
  pool: zvol0
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Sat Jan 28 14:25:44 2023
        447G scanned at 140M/s, 196G issued at 61.4M/s, 1.49T total
        196G resilvered, 12.85% done, 06:09:14 to go
config:

        NAME                        STATE     READ WRITE CKSUM
        zvol0                       DEGRADED     0     0     0
          mirror-0                  DEGRADED     0     0     0
            replacing-0             DEGRADED     0     0     0
              12897545936258916783  UNAVAIL      0     0     0  was /dev/gpt/ndisk2
              gpt/sdisk1            ONLINE       0     0     0  (resilvering)
            gpt/ndisk1              ONLINE       0     0     0

errors: No known data errors
$
```

結局15時間以上かかって終了

```
$ zpool list -v
NAME             SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot            228G  16.8G   211G        -         -    12%     7%  1.00x    ONLINE  -
  ada0p3         228G  16.8G   211G        -         -    12%  7.37%      -    ONLINE
zvol0           1.81T  1.49T   333G        -         -     5%    82%  1.00x    ONLINE  -
  mirror-0      1.81T  1.49T   333G        -         -     5%  82.1%      -    ONLINE
    gpt/sdisk1      -      -      -        -         -      -      -      -    ONLINE
    gpt/ndisk1      -      -      -        -         -      -      -      -    ONLINE
$ zpool status zvol0
  pool: zvol0
 state: ONLINE
  scan: resilvered 1.49T in 15:34:45 with 0 errors on Sun Jan 29 06:00:29 2023
config:

        NAME            STATE     READ WRITE CKSUM
        zvol0           ONLINE       0     0     0
          mirror-0      ONLINE       0     0     0
            gpt/sdisk1  ONLINE       0     0     0
            gpt/ndisk1  ONLINE       0     0     0

errors: No known data errors
$
```

 残りのディスクを交換

```
$ zpool status zvol0
  pool: zvol0
 state: DEGRADED
status: One or more devices could not be opened.  Sufficient replicas exist for
        the pool to continue functioning in a degraded state.
action: Attach the missing device and online it using 'zpool online'.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-2Q
  scan: resilvered 1.49T in 15:34:45 with 0 errors on Sun Jan 29 06:00:29 2023
config:

        NAME                      STATE     READ WRITE CKSUM
        zvol0                     DEGRADED     0     0     0
          mirror-0                DEGRADED     0     0     0
            gpt/sdisk1            ONLINE       0     0     0
            13953654332474917058  UNAVAIL      0     0     0  was /dev/gpt/ndisk1

errors: No known data errors
$
```

ディスクを用意

```
# gpart create -s gpt ada2
ada2 created
# gpart show ada2
=>         40  11721045088  ada2  GPT  (5.5T)
           40  11721045088        - free -  (5.5T)

# gpart add -t freebsd-zfs -a 4k -l sdisk2 ada2
ada2p1 added
# gpart show ada2
=>         40  11721045088  ada2  GPT  (5.5T)
           40  11721045088     1  freebsd-zfs  (5.5T)

# gpart show -l ada2
=>         40  11721045088  ada2  GPT  (5.5T)
           40  11721045088     1  sdisk2  (5.5T)

#
```

```
$ zpool replace zvol0 13953654332474917058 gpt/sdisk2
$ zpool status zvol0
  pool: zvol0
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Sun Jan 29 13:38:15 2023
        12.8G scanned at 692M/s, 492K issued at 25.9K/s, 1.49T total
        0B resilvered, 0.00% done, no estimated completion time
config:

        NAME                        STATE     READ WRITE CKSUM
        zvol0                       DEGRADED     0     0     0
          mirror-0                  DEGRADED     0     0     0
            gpt/sdisk1              ONLINE       0     0     0
            replacing-1             DEGRADED     0     0     0
              13953654332474917058  UNAVAIL      0     0     0  was /dev/gpt/ndisk1
              gpt/sdisk2            ONLINE       0     0     0

errors: No known data errors
```

多分2回目の同期は1回目よりはかなり短時間になると予想しているが如何に


```
$ zpool status zvol0
  pool: zvol0
 state: ONLINE
  scan: resilvered 1.49T in 14:11:00 with 0 errors on Mon Jan 30 03:49:15 2023
config:

        NAME            STATE     READ WRITE CKSUM
        zvol0           ONLINE       0     0     0
          mirror-0      ONLINE       0     0     0
            gpt/sdisk1  ONLINE       0     0     0
            gpt/sdisk2  ONLINE       0     0     0

errors: No known data errors
$
```
結局 14時間程かかって終了したようです。
```
$ zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zroot   228G  16.8G   211G        -         -    12%     7%  1.00x    ONLINE  -
zvol0  1.81T  1.49T   333G        -     3.62T     5%    82%  1.00x    ONLINE  -
$
```


領域の拡張

```
$ zpool list zvol0
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zvol0  1.81T  1.49T   333G        -     3.62T     5%    82%  1.00x    ONLINE  -
$
$ zpool get autoexpand zvol0
NAME   PROPERTY    VALUE   SOURCE
zvol0  autoexpand  off     default
$
$ zpool set autoexpand=on zvol0
$
$ zpool get autoexpand zvol0
NAME   PROPERTY    VALUE   SOURCE
zvol0  autoexpand  on      local
$
$ zpool list zvol0
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zvol0  1.81T  1.49T   333G        -     3.62T     5%    82%  1.00x    ONLINE  -
$
$ zpool online -e zvol0
missing device name
usage:
        online [-e] <pool> <device> ...
$
$ zpool online -e zvol0 gpt/sdisk1
$
$ zpool list zvol0
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zvol0  5.45T  1.49T  3.97T        -         -     1%    27%  1.00x    ONLINE  -
$
```


参考データ
```
minmin@primergy(0)$ zpool status zvol0
  pool: zvol0
 state: ONLINE
  scan: resilvered 1.49T in 06:49:36 with 0 errors on Mon Jan 30 17:58:41 2023
config:

        NAME            STATE     READ WRITE CKSUM
        zvol0           ONLINE       0     0     0
          mirror-0      ONLINE       0     0     0
            gpt/ndisk2  ONLINE       0     0     0
            gpt/ndisk1  ONLINE       0     0     0

errors: No known data errors
minmin@primergy(0)$
```