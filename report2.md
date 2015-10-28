#レポート課題 OpenFlow1.3
> OpenFlow1.3版のスイッチの動作について，  
> `trema dump_flows`の出力を混じえながら各ステップごとに説明しよう．    

> 実行のしかた  
> `$ trema run lib/learning_switch13.rb --openflow13 -c trema.conf`  

## 動作説明
### テーブルの初期化
スイッチがコントローラに接続されると`switch_ready`ハンドラが呼び出されフローテーブルが初期化される．  
フローテーブルの初期化内容は以下の通りである．

```
【Filteringテーブル (Table ID: 0)】
優先度 2 : 宛先MACアドレスがマルチキャストMACアドレス('01:00:5e:XX:XX:XX')であればDropする … ルールD (Drop)
優先度 1 : GoTo Fowardingテーブル (Table ID: 1) … ルールG (GoTo)
```

```
【Fowardingテーブル (Table ID: 1)】
優先度 1 : コントローラにPortOutし，パケットをFloodingする … ルールP (PacketIn)
```

### PacketInが発生したとき
PacketInが発生すると`packet_in`ハンドラが呼び出される．  
そのとき，Fowardingテーブルに以下のルールを追加する．

```
優先度: 1
ルール破棄時間: 180秒 (AGING_TIME)
マッチングルール: 送信先MAC が パケットの送信元MAC
動作: パケットの送信元ポート番号にパケットを転送
```

## 動作確認
### 想定環境
スイッチを1つ(lsw)を用意し，そのスイッチにホストを2つ接続し(host1, host2)動作確認を行った．  
confファイルの詳細を以下に示す．

```
【confファイル】
vswitch('lsw') {
  datapath_id 0xdef
}

vhost ('host1') {
  ip '192.168.0.1'
  mac '00:00:00:00:00:01'
}

vhost ('host2') {
  ip '192.168.0.2'
  mac '00:00:00:00:00:02'
}


link 'lsw', 'host1'
link 'lsw', 'host2'
```

### 動作テスト
#### host1とhost2のパケット送受信
##### 予想される動作
**host1**と**host2**のパケット送受信のテストをした．  
**host1**から**host2**へパケットを送信する際に予想される動作を以下に述べる．

1. **host1**から**lsw**にパケットが転送される．  
2. Filteringテーブルで送信先(**host2**)のMACアドレスが`00:00:00:00:00:02`なので，**ルールD**が適用されない為，パケットがDropされずに**ルールG**が適用される．  
3. Fowardingテーブルで**ルールP**が適用され，コントローラにPacketInを送信し，パケットをフラッディングする．  
4. 【コントローラ】PacketInを受信したコントローラは，宛先のMACアドレスが**host1**(`00:00:00:00:00:01`)のパケットの転送先を**host1**のポート番号(ポート1)に転送するルールをFowardingテーブルに登録する．
5. **host2**は**lsw**からフラッディングされたパケットを受信する．

処理後，スイッチ**lsw**のフローテーブルは以下のようになっているはずである．

```
【Filteringテーブル (Table ID: 0)】
優先度 2 : 宛先MACアドレスがマルチキャストMACアドレス('01:00:5e:XX:XX:XX')であればDropする … ルールD (Drop)
優先度 1 : GoTo Fowardingテーブル (Table ID: 1) … ルールG (GoTo)
```

```
【Fowardingテーブル (Table ID: 1)】
優先度 2 : 宛先MACアドレスがhost1(`00:00:00:00:00:01`)であれば，パケットをポート1にPortOutする． … ルール1 (host1)
優先度 1 : コントローラにPortOutし，パケットをFloodingする … ルールP (PacketIn)
```

さらに続けて，**host2**から**host1**へパケットを送信する際に予想される動作を以下に述べる．

1. **host2**から**lsw**にパケットが転送される．
2. Filteringテーブルで送信先(**host2**)のMACアドレスが`00:00:00:00:00:02`なので，**ルールD**が適用されない為，パケットがDropされずに**ルールG**が適用される．  
3. Fowardingテーブルで**ルール1**が適用され，パケットをポート1(**host1**)にPortOutする．
4. **host1**は**lsw**からPortOutされたパケットを受信する． 


##### 実行結果
動作テストの実行結果を以下に示す．  
まず，起動直後の**lsw**のフローテーブルを以下に示す．  
上から順に**ルールD**，**ルールG**，**ルールP**が設定されていることが確認できる．

```
【起動直後のフローテーブル】
$ sudo ovs-ofctl dump-flows brlsw --protocol=OpenFlow13
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=3.309s, table=0, n_packets=0, n_bytes=0, priority=2,
dl_dst=01:00:5e:00:00:00/ff:ff:ff:00:00:00 actions=drop
 cookie=0x0, duration=3.271s, table=0, n_packets=0, n_bytes=0, priority=1 actions=goto_table:1
 cookie=0x0, duration=3.271s, table=1, n_packets=0, n_bytes=0, priority=1 actions=CONTROLLER:65535,FLOOD
```

そして，**host1**から**host2**へパケットを送信し，送信後のフローテーブルを確認した．  
**host1**で送信されたパケットは正常に**host2**で受信できていた．  
また，フローテーブル(`Fowardingテーブル`)に新しいルールが追加されていることが確認できる．  
新たに追加されたルールは宛先MACアドレスが`00:00:00:00:00:01`のパケットをポート1(**host1**)に転送するといったルールになっている．

```
$ bin/trema send_packet --source host1 --dest host2
$ bin/trema show_stats host1
Packets sent:
  192.168.0.1 -> 192.168.0.2 = 1 packet
$ bin/trema show_stats host2
Packets received:
  192.168.0.1 -> 192.168.0.2 = 1 packet
$ sudo ovs-ofctl dump-flows brlsw --protocol=OpenFlow13
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=22.424s, table=0, n_packets=0, n_bytes=0, priority=2,
dl_dst=01:00:5e:00:00:00/ff:ff:ff:00:00:00 actions=drop
 cookie=0x0, duration=22.386s, table=0, n_packets=1, n_bytes=42, priority=1 actions=goto_table:1
 cookie=0x0, duration=10.115s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,
dl_dst=00:00:00:00:00:01 actions=output:1
 cookie=0x0, duration=22.386s, table=1, n_packets=1, n_bytes=42, priority=1 actions=CONTROLLER:65535,FLOOD
```

その後，**host2**から**host1**へパケットを送信し，送信後のフローテーブルを確認した．  
**host2**で送信されたパケットは正常に**host1**で受信できていた．   
また，前述のとおりフローテーブル(`Fowardingテーブル`)にルールが追加されていないことが確認できる．

```
$ bin/trema send_packet --source host2 --dest host1
$ bin/trema show_stats host1
Packets sent:
  192.168.0.1 -> 192.168.0.2 = 1 packet
Packets received:
  192.168.0.2 -> 192.168.0.1 = 1 packet
$ bin/trema show_stats host2
Packets sent:
  192.168.0.2 -> 192.168.0.1 = 1 packet
Packets received:
  192.168.0.1 -> 192.168.0.2 = 1 packet
$ sudo ovs-ofctl dump-flows brlsw --protocol=OpenFlow13
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=52.24s, table=0, n_packets=0, n_bytes=0, priority=2,
dl_dst=01:00:5e:00:00:00/ff:ff:ff:00:00:00 actions=drop
 cookie=0x0, duration=52.202s, table=0, n_packets=2, n_bytes=84, priority=1 actions=goto_table:1
 cookie=0x0, duration=39.931s, table=1, n_packets=1, n_bytes=42, idle_timeout=180, priority=2,
dl_dst=00:00:00:00:00:01 actions=output:1
 cookie=0x0, duration=52.202s, table=1, n_packets=1, n_bytes=42, priority=1 actions=CONTROLLER:65535,FLOOD
```


## 考察
### 今回の課題のコントローラの不具合
前回の課題で利用したコントローラは，PacketInされたときに，パケットの宛先MACアドレスがFDBに登録済みであれば，フローテーブルにルールを追加するといった処理であった．
そのため，フローテーブルにルールを追加されるのは**host1 → hostA** (**hostA**は任意のホスト)へパケットが送信され，その後 **hostA → host1** (**hostA**は任意のホスト)へパケットが送信された時に初めてフローテーブルに**host1**への転送に関するルールが追加されていた．
ようするに，フローテーブルにルールを追加するのにパケットの送信が2回必要であった．

今回の課題で利用したコントローラは，パケットが送信され**PacketInが発生したとき**にパケットの送信元MACアドレスと送信元ポート番号からフローテーブルにルールを追加している．
そのため，**host1** → **hostA**へパケットが送信された時にフローテーブルに**host1**への転送に関するルールが追加されている．
しかし，この手法だと続けて**host2** → **host1**へパケットを送信した時に，送信先の**host1**がフローテーブルに登録済みであるため，PacketInが発生しないのでフローテーブルは更新されずに**host1**のポートへPortOutされる．

***この手法だとここで問題が発生する***．
再度，**host1 → host2**へパケットを送信した時を考えると，送信先の**host2**はフローテーブルに登録されていないため，PacketInが発生する．
このコントローラは前述のとおりPacketInが発生した時にフローテーブルにルールを追加するが，このとき追加するルールはパケットの送信元であるhost1に関する転送のルールである．**host1**への転送のルールは**host1 → hostA**へ転送した時に既に設定済みであるため，実質フローテーブルは更新されない．
ということは，ようするに，**host1**と**host2**でパケットの送受信としているうちは，**host1**に関する転送のルールは設定されるが**host2**に関する転送のルールは***永遠に設定されない***ということになる．
よって，今回の課題で利用したコントローラは，**host1 → host2**へ送信されるパケットは***常にPacketInが発生***する．
そのため，___コントローラに負荷を与え，さらにスループットが低下するといった不具合___があると考えられる．
