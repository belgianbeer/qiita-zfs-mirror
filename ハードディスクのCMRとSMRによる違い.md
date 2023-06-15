# ハードディスクのCMRとSMRによるパフォーマンスの違い

ZFSミラープールのディスク交換による容量増設という記事を書きました。交換した新しい6TBのハードディスクの記録方式はSMRで、それまで使っていた10年以上前の2TBのハードディスクはCMRでした。詳細は説明しませんが、SMRではその構造上大量データの連続書き込みがCMRに比べてどうしても遅くなる傾向があります(読み出しはSMRとCMRによる違いはありません)。

ミラーの同期は大量データの連続書き込みであるため、SMRの苦手な処理の典型的な例ということになります。そこで実際にCMRの古いハードディスクと、SMRの新しいハードディスクのミラーの同期にかかる時間を比較してみます。

## SMRでの同期時間

こちらが該当記事に掲載したSMRのハードディスクでのミラーの同期にかかった時間の記録で、14時間11分となっています。

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

## CMRでの同期時間

交換前の2TBのハードディスクにも同じデータが残っているため、改めて同期にかかる時間を計測してみました。結果は次に示すように6時間50分程度と、SMRのものに比べるとかなり短い時間となりました[^big]。

[^big]: ZFSのRAIDの同期では書き込み済みの領域だけを同期するため、ハードディスクの総容量の違いは関係ありません。

```
$ zpool status zvol0
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
$
```

## 実際に使ってみた違い

ファイルサーバーではあるもののユーザーは自分1人です。その状況で実際にSMRのハードディスクへ交換後に使ってみた体感では、日常的に使う分にはSMRのハードディスクによる書き込み時の速度低下を感じないどころか、以前のものより速くなった感じすらあります。速くなったと感じる一つの要因は、ハードディスクが世代的に新しくなって記録密度が向上し転送速度が上がっていることでしょう。そして筆者の日常利用の書き込み程度であれば、SMRの欠点はハードディスクが持つバッファメモリとOSのバッファによって隠されてしまうため体感することは難しそうです。

## まとめ

大量書き込み時に遅いからとSMRのハードディスクを毛嫌いする必要は無さそうですが、RAIDの同期時間を短時間で済ませたい場合等には、SMRは避けてCMRのハードディスクを選ぶほうが良さそうです。
