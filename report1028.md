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
優先度 2 : 宛先MACアドレスがマルチキャストMACアドレス('01:00:00:00:00:00/FF:00:00:00:00:00')であればDropする … エントリDM (Drop Multicast)
優先度 2 : 宛先MACアドレスがipv6マルチキャストMACアドレス('33:33:00:00:00:00/FF:FF:00:00:00:00')であればDropする … エントリD6 (Drop IPv6 Multicast)
優先度 1 : GoTo Fowardingテーブル (Table ID: 1) … エントリG (GoTo)
```

```
【Fowardingテーブル (Table ID: 1)】
優先度 3 : ブロードキャストであればフラッディングをする … エントリB (Bloadcast)
優先度 1 : コントローラにPacketInする … エントリP (PacketIn)
```

### PacketInが発生したとき
PacketInが発生すると`packet_in`ハンドラが呼び出され，必要であればFDBを更新する処理がなされる．
そのときFDBは送信元MACアドレスと送信元ポート番号の学習を行う．  
そして，`add_forwarding_flow_and_packet_out`メソッドを呼び出しFlowModやPacketOutを行っている．  
パケットの宛先MACアドレスが既にFDBに登録済みであれば，Fowardingテーブルに以下のフローエントリを追加し，そのポート番号へPacketOutする．  
FDBに登録されていなければ，パケットをフラッディングとしてPacketOutする．
```
【追加フローエントリ】
優先度: 2
エントリ破棄時間: 180秒 (AGING_TIME)
マッチングルール: 当該パケットの送信元ポート番号，宛先MACアドレス，送信元MACアドレスが完全に一致
動作: パケットの送信先ポート番号にパケットをSendOutPort
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
2. Filteringテーブルで送信先(**host2**)のMACアドレスが`00:00:00:00:00:02`なので，**エントリDM, エントリD6**が適用されない為，パケットがDropされずに**エントリG**が適用される．  
3. Fowardingテーブルで**エントリP**が適用され，PacketInが発生する．
4. PacketInを受信したコントローラは，FDBが空なのでフローエントリは追加せずに，パケットをフラッディングとしてPackeOutをする．また，FDBに送信元MACアドレス(`00:00:00:00:00:01`)と送信元ポート番号(ポート1)を登録する．
5. **host2**は**lsw**からフラッディングされたパケットを受信する．

処理後，スイッチ**lsw**のフローテーブルに変化はない．


さらに続けて，**host2**から**host1**へパケットを送信する際に予想される動作を以下に述べる．

1. **host2**から**lsw**にパケットが転送される．
2. Filteringテーブルで送信先(**host1**)のMACアドレスが`00:00:00:00:00:01`なので，**エントリDM, エントリD6**が適用されない為，パケットがDropされずに**エントリG**が適用される．  
3. Fowardingテーブルで**エントリP**が適用され，PacketInが発生する．
4. PacketInを受信したコントローラは，FDBに宛先MACアドレス(`00:00:00:00:00:01`)が登録済みなので，フローエントリを追加し，パケットをhost1(ポート1)へPacketOutする．
5. **host1**は**lsw**からPacketOutされたパケットを受信する． 

処理後，スイッチ**lsw**のフローテーブルは以下のようになっているはずである．

```
【Filteringテーブル (Table ID: 0)】
優先度 2 : 宛先MACアドレスがマルチキャストMACアドレス('01:00:00:00:00:00/FF:00:00:00:00:00')であればDropする … エントリDM (Drop Multicast)
優先度 2 : 宛先MACアドレスがipv6マルチキャストMACアドレス('33:33:00:00:00:00/FF:FF:00:00:00:00')であればDropする … エントリD6 (Drop IPv6 Multicast)
優先度 1 : GoTo Fowardingテーブル (Table ID: 1) … エントリG (GoTo)
```

```
【Fowardingテーブル (Table ID: 1)】
優先度 3 : ブロードキャストであればフラッディングをする … エントリB (Bloadcast)
優先度 2 : 送信元ポート番号 = 2，送信元MACアドレス = 00:00:00:00:00:02，宛先MACアドレス = 00:00:00:00:00:01 であれば，ポート1へ転送する． … エントリ1 (host1)
優先度 1 : コントローラにPacketInする … エントリP (PacketIn)
```

再度，host1からhost2へパケットを送信する際に予想される動作を以下に述べる．
1. **host1**から**lsw**にパケットが転送される．
2. Filteringテーブルで送信先(**host2**)のMACアドレスが`00:00:00:00:00:02`なので，**エントリDM, エントリD6**が適用されない為，パケットがDropされずに**エントリG**が適用される．  
3. Fowardingテーブルで**エントリP**が適用され，PacketInが発生する．
4. PacketInを受信したコントローラは，FDBに宛先MACアドレス(`00:00:00:00:00:02`)が登録済みなので，フローエントリを追加し，パケットをhost2(ポート2)へPacketOutする．
5. **host2**は**lsw**からPacketOutされたパケットを受信する． 

処理後，スイッチ**lsw**のフローテーブルは以下のようになっているはずである．

```
【Filteringテーブル (Table ID: 0)】
優先度 2 : 宛先MACアドレスがマルチキャストMACアドレス('01:00:00:00:00:00/FF:00:00:00:00:00')であればDropする … エントリDM (Drop Multicast)
優先度 2 : 宛先MACアドレスがipv6マルチキャストMACアドレス('33:33:00:00:00:00/FF:FF:00:00:00:00')であればDropする … エントリD6 (Drop IPv6 Multicast)
優先度 1 : GoTo Fowardingテーブル (Table ID: 1) … エントリG (GoTo)
```

```
【Fowardingテーブル (Table ID: 1)】
優先度 3 : ブロードキャストであればフラッディングをする … エントリB (Bloadcast)
優先度 2 : 送信元ポート番号 = 2，送信元MACアドレス = 00:00:00:00:00:02，宛先MACアドレス = 00:00:00:00:00:01 であれば，ポート1へ転送する． … エントリ1 (host1)
優先度 2 : 送信元ポート番号 = 1，送信元MACアドレス = 00:00:00:00:00:01，宛先MACアドレス = 00:00:00:00:00:02 であれば，ポート2へ転送する． … エントリ2 (host2)
優先度 1 : コントローラにPacketInする … エントリP (PacketIn)
```


##### 実行結果
動作テストの実行結果を以下に示す．  
まず，起動直後の**lsw**のフローテーブルを以下に示す．  
上から順に**エントリDM**，**エントリD6**，**エントリG**，**エントリB**，**エントリP**が設定されていることが確認できる．

```
【起動直後のフローテーブル】
$ sudo ovs-ofctl dump-flows brlsw --protocol=OpenFlow13OFPST_FLOW reply (OF1.3) (xid=0x2): cookie=0x0, duration=48.023s, table=0, n_packets=0, n_bytes=0, 
priority=2, dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop cookie=0x0, duration=47.985s, table=0, n_packets=0, n_bytes=0, priority=2, dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop cookie=0x0, duration=47.973s, table=0, n_packets=0, n_bytes=0, 
priority=1 actions=goto_table:1 cookie=0x0, duration=47.985s, table=1, n_packets=0, n_bytes=0, 
priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD cookie=0x0, duration=47.981s, table=1, n_packets=0, n_bytes=0, 
priority=1 actions=CONTROLLER:65535```

そして，**host1**から**host2**へパケットを送信し，送信後のフローテーブルを確認した．  
**host1**で送信されたパケットは正常に**host2**で受信できていた．   
また，フローテーブルは更新されておらず，予想通りの動作が行われていることが確認できる．

```
$ bin/trema send_packet -s host1 -d host2$ bin/trema show_stats host1Packets sent:  192.168.0.1 -> 192.168.0.2 = 1 packet$ bin/trema show_stats host2Packets received:  192.168.0.1 -> 192.168.0.2 = 1 packet$ sudo ovs-ofctl dump-flows brlsw --protocol=OpenFlow13OFPST_FLOW reply (OF1.3) (xid=0x2): cookie=0x0, duration=69.18s, table=0, n_packets=0, n_bytes=0, 
priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop cookie=0x0, duration=69.142s, table=0, n_packets=0, n_bytes=0, 
priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop cookie=0x0, duration=69.13s, table=0, n_packets=1, n_bytes=42, 
priority=1 actions=goto_table:1 cookie=0x0, duration=69.142s, table=1, n_packets=0, n_bytes=0, 
priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD cookie=0x0, duration=69.138s, table=1, n_packets=1, n_bytes=42, 
priority=1 actions=CONTROLLER:65535```

その後，**host2**から**host1**へパケットを送信し，送信後のフローテーブルを確認した．  
**host2**で送信されたパケットは正常に**host1**で受信できていた．   
また，フローテーブル(`Fowardingテーブル`)に新しいフローエントリが追加されていることが確認できる．    
新たに追加されたフローエントリは宛先MACアドレス = 00:00:00:00:00:01，送信元MACアドレス = 00:00:00:00:00:02，送信元ポート番号 = 2のパケットをポート1(host1)に転送するといったフローエントリになっている．


```
$ bin/trema send_packet -s host2 -d host1$ bin/trema show_stats host1Packets sent:  192.168.0.1 -> 192.168.0.2 = 1 packetPackets received:  192.168.0.2 -> 192.168.0.1 = 1 packet$ bin/trema show_stats host2Packets sent:  192.168.0.2 -> 192.168.0.1 = 1 packetPackets received:  192.168.0.1 -> 192.168.0.2 = 1 packet$ sudo ovs-ofctl dump-flows brlsw --protocol=OpenFlow13OFPST_FLOW reply (OF1.3) (xid=0x2): cookie=0x0, duration=86.502s, table=0, n_packets=0, n_bytes=0, 
priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop cookie=0x0, duration=86.464s, table=0, n_packets=0, n_bytes=0, 
priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop cookie=0x0, duration=86.452s, table=0, n_packets=2, n_bytes=84, 
priority=1 actions=goto_table:1 cookie=0x0, duration=86.464s, table=1, n_packets=0, n_bytes=0, 
priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD cookie=0x0, duration=8.932s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, 
priority=2,in_port=2,dl_src=4b:5b:b9:68:a4:c9,dl_dst=ff:05:3f:a8:9a:70 actions=output:1 cookie=0x0, duration=86.46s, table=1, n_packets=2, n_bytes=84,
priority=1 actions=CONTROLLER:65535```

再度，**host1**から**host2**へパケットを送信し，送信後のフローテーブルを確認した．  
**host1**で送信されたパケットは正常に**host2**で受信できていた．   
また，フローテーブル(`Fowardingテーブル`)に新しいフローエントリが追加されていることが確認できる．    
新たに追加されたフローエントリは宛先MACアドレス = 00:00:00:00:00:02，送信元MACアドレス = 00:00:00:00:00:01，送信元ポート番号 = 1のパケットをポート2(host2)に転送するといったフローエントリになっている．

```
$ bin/trema send_packet -s host1 -d host2$ bin/trema show_stats host1Packets sent:  192.168.0.1 -> 192.168.0.2 = 2 packetsPackets received:  192.168.0.2 -> 192.168.0.1 = 1 packet$ bin/trema show_stats host2Packets sent:  192.168.0.2 -> 192.168.0.1 = 1 packetPackets received:  192.168.0.1 -> 192.168.0.2 = 2 packets$ sudo ovs-ofctl dump-flows brlsw --protocol=OpenFlow13OFPST_FLOW reply (OF1.3) (xid=0x2): cookie=0x0, duration=104.252s, table=0, n_packets=0, n_bytes=0, priority=2,
dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop cookie=0x0, duration=104.214s, table=0, n_packets=0, n_bytes=0, priority=2,
dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop cookie=0x0, duration=104.202s, table=0, n_packets=3, n_bytes=126, priority=1 actions=goto_table:1 cookie=0x0, duration=104.214s, table=1, n_packets=0, n_bytes=0, priority=3,
dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD cookie=0x0, duration=26.682s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=2,
dl_src=4b:5b:b9:68:a4:c9,dl_dst=ff:05:3f:a8:9a:70 actions=output:1 cookie=0x0, duration=8.542s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=1,
dl_src=ff:05:3f:a8:9a:70,dl_dst=4b:5b:b9:68:a4:c9 actions=output:2 cookie=0x0, duration=104.21s, table=1, n_packets=3, n_bytes=126, priority=1 actions=CONTROLLER:65535
```


## 考察
### ~~今回の課題の~~修正前のコントローラの不具合
前回の課題で利用したコントローラは，PacketInされたときに，パケットの宛先MACアドレスがFDBに登録済みであれば，フローテーブルにフローエントリを追加するといった処理であった．
そのため，フローテーブルにフローエントリを追加されるのは**host1 → hostA** (**hostA**は任意のホスト)へパケットが送信され，その後 **hostA → host1** (**hostA**は任意のホスト)へパケットが送信された時に初めてフローテーブルに**host1**への転送に関するルールが追加されていた．
ようするに，フローテーブルにフローエントリを追加するのにパケットの送信が2回必要であった．

~~今回の課題で利用した~~修正前のコントローラは，パケットが送信され**PacketInが発生したとき**にパケットの送信元MACアドレスと送信元ポート番号からフローテーブルにフローエントリを追加している．
そのため，**host1** → **hostA**へパケットが送信された時にフローテーブルに**host1**への転送に関するフローエントリが追加されている．
しかし，この手法だと続けて**host2** → **host1**へパケットを送信した時に，送信先の**host1**がフローテーブルに登録済みであるため，PacketInが発生しないのでフローテーブルは更新されずに**host1**のポートへPortOutされる．

***この手法だとここで問題が発生する***．
再度，**host1 → host2**へパケットを送信した時を考えると，送信先の**host2**はフローテーブルに登録されていないため，PacketInが発生する．
このコントローラは前述のとおりPacketInが発生した時にフローテーブルにフローエントリを追加するが，このとき追加するフローエントリはパケットの送信元であるhost1に関する転送のフローエントリである．**host1**への転送のルールは**host1 → hostA**へ転送した時に既に設定済みであるため，実質フローテーブルは更新されない．
ということは，ようするに，**host1**と**host2**でパケットの送受信としているうちは，**host1**に関する転送のフローエントリは設定されるが，**host2**に関する転送のフローエントリは***永遠に設定されない***ということになる．

よって，~~今回の課題で利用した~~修正前のコントローラは，**host1 → host2**へ送信されるパケットは***常にPacketInが発生***する．
そのため，___コントローラに負荷を与え，さらにスループットが低下するといった不具合___があると考えられる．
