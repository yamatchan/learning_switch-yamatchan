#learning_switchレポート課題
##複数スイッチ対応版(multi_learning_switch.rb)の動作を解説せよ
### startメソッド
それぞれのスイッチにFDBが存在するので，FDBは`@fdbs`に連想配列として管理し，複数のFDBオブジェクトが格納できるようにしている．  
コントローラを起動し，`start`ハンドラが呼びだされたときに，`@fdbs`をハッシュで初期化している(10行目)．

```
 9   def start(_argv)
10     @fdbs = {}
11     logger.info 'MultiLearningSwitch started.'
12   end
```

### switch_readyメソッド
スイッチがコントローラに接続されると`switch_ready`ハンドラが呼び出され，
連想配列`@fdbs`にスイッチのDatapathId(`datapath_id`)をキーとして，新たに生成したFDBオブジェクトを格納している(15行目)．

```
14   def switch_ready(datapath_id)
15     @fdbs[datapath_id] = FDB.new
16   end
```

### packet_inメソッド
PacketInが発生すると`packet_in`ハンドラが呼び出され，必要であれば当該スイッチのFDBを更新する処理がなされる．  
FDBは複数存在するのでFDBを更新する際，`@fdbs.fetch(datapath_id)`でPacketInが発生したスイッチのFDBオブジェクトを参照し，MACアドレスとポート番号の学習を行う．  
そして，`flow_mod_and_packet_out`メソッドを呼び出しFlowModやPacketOutを行っている．

### flow_mod_and_packet_outメソッド
送信先のMACアドレスをポート番号に変換する必要があるので，
`@fdbs.fetch(message.dpid).lookup(message.destination_mac)`でFDBに登録されているMACアドレスを参照する．  
* FDBにMACアドレスが登録されているとき  
当該スイッチのフローテーブルにフローエントリを追加し，送信先のMACアドレスが接続しているポート番号に向けてPacketOutする．  
* DBにMACアドレスが登録されていないとき  
送信先のポート番号が分からないのでフローテーブルは更新せずに，`:flood`オプションを付加してPacketOutする．  
そのPacketOutを受け取ったスイッチは指定されたパケットをフラッディングし，目的のホストにパケットを送信する．  


### 動作確認
#### 想定環境
スイッチを4つ(lsw1, lsw2, lsw3, lws4)を用意し，それぞれのスイッチにホストを2つずつ接続し(lsw1; host1-1/host1-2, lsw2; host2-1/host2-2, lsw3; host3-1/host3-1, lsw4l host4-1/host4-2)動作確認を行った．  
接続のイメージ図とconfファイルの詳細を以下に示す．

![想定環境イメージ図](https://github.com/handai-trema/learning_switch-yamatchan/blob/master/img/img1.png)

```
vswitch('lsw1') { datapath_id 0x1 }
vswitch('lsw2') { datapath_id 0x2 }
vswitch('lsw3') { datapath_id 0x3 }
vswitch('lsw4') { datapath_id 0x4 }

vhost('host1-1') {
 ip '192.168.0.11'
 mac '00:00:00:00:00:11'
}
vhost('host1-2') {
 ip '192.168.0.12'
 mac '00:00:00:00:00:12'
}
vhost('host2-1') {
 ip '192.168.0.21'
 mac '00:00:00:00:00:21'
}
vhost('host2-2') {
 ip '192.168.0.22'
 mac '00:00:00:00:00:22'
}
vhost('host3-1') {
 ip '192.168.0.31'
 mac '00:00:00:00:00:31'
}
vhost('host3-2') {
 ip '192.168.0.32'
 mac '00:00:00:00:00:32'
}
vhost('host4-1') {
 ip '192.168.0.41'
 mac '00:00:00:00:00:41'
}
vhost('host4-2') {
 ip '192.168.0.42'
 mac '00:00:00:00:00:42'
}

link 'lsw1', 'host1-1'
link 'lsw1', 'host1-2'
link 'lsw2', 'host2-1'
link 'lsw2', 'host2-2'
link 'lsw3', 'host3-1'
link 'lsw3', 'host3-2'
link 'lsw4', 'host4-1'
link 'lsw4', 'host4-2'
```

#### 動作テスト その1
##### host1-1とhost1-2の送受信
###### 予想される動作
**host1-1**と**host1-2**の送受信のテストをした．  
**host1-1**から**host1-2**へパケットを送信する際に予想される動作を以下に述べる．(MACアドレスはコロンを含めた末尾3桁に省略した形で記述する)  

1. **host1-1**から**lsw1**へパケットが転送される
2. **lsw1**のフローテーブルが空なので，PacketInがコントローラに送られる． 
3. コントローラはPacketInメッセージから**host1-1**のMACアドレスとポート番号をFDB1に保存する．[FDB1; ***:11 => ポート1***]   
4. コントローラは送信先の**host1-2**のMACアドレスがFDB1に登録されていないので，パケットをフラッディングとしてPacketOutする．  
5. PacketOutを受け取った**lsw1**はパケットをフラッディングするので，結果，パケットが**host1-2**へ転送される．
6. **host1-2**が**lsw1**からパケットを受け取る．  

![host1-1からhost1-2へ送信したときの挙動](https://github.com/handai-trema/learning_switch-yamatchan/blob/master/img/img2.png)

※処理後，全てのスイッチのフローテーブルは更新されていないので空のままのはずである．  
<br />

続けて，**host1-2**から**host1-1**へパケットを送信する際に予想される動作を以下に述べる．

1. **host1-2**から**lsw1**へパケットが転送される．
2. **lsw1**のフローテーブルが空なので，PacketInがコントローラに送られる．
3. コントローラはPacketInメッセージから**host1-2**のMACアドレスとポート番号をFDB1に保存する．[FDB1; :11 => ポート1, ***:12 => ポート2***]  
さらに，送信先の**host1-1**のMACアドレスがFDB1に登録されているので，フローテーブルFT1を更新する．[FT1; ***:12->:11 => ポート1へ***]  
4. コントローラは送信先の**host1-1**が接続されているポート番号が分かっているので，送信ポート番号(ポート1)を付加してPacketOutする．
5. PacketOutを受け取った**lsw1**はパケットをポート1(**host1-1**)へパケットを転送する．
6. **host1-1**が**lsw1**からパケットを受け取る．  

![host1-2からhost1-1へ送信したときの挙動](https://github.com/handai-trema/learning_switch-yamatchan/blob/master/img/img3.png)

処理後，それぞれのスイッチのフローテーブルは以下のようになっているはずである．

```
lsw1: :12->:11 => ポート1へ (host1-1)
lsw2: 空
lsw3: 空
lsw4: 空
```
<br />

さらに続けて，**host1-1**から**host1-2**へパケットを送信する際に予想される動作を以下に述べる．

1. **host1-1**から**lsw1**へパケットが転送される．
2. **lsw1**のフローテーブルに条件がマッチングするものがないので，PacketInがコントローラに送られる．
3. 送信先の**host1-2**のMACアドレスがFDB1に登録されているので，フローテーブルFT1を更新する．[FT1; :12->:11 => ポート1へ, ***:11->:12 => ポート2へ***]  
4. コントローラは送信先の**host1-2**が接続されているポート番号が分かっているので，送信ポート番号(ポート2)を付加してPacketOutする．
5. PacketOutを受け取った**lsw1**はパケットをポート2(**host1-2**)へパケットを転送する．
6. **host1-2**が**lsw1**からパケットを受け取る．  

![host1-1からhost1-2へ再度送信したときの挙動](https://github.com/handai-trema/learning_switch-yamatchan/blob/master/img/img4.png)

処理後，それぞれのスイッチのフローテーブルは以下のようになっているはずである．

```
lsw1: :12->:11 => ポート1へ (host1-1)，:11->:12 => ポート2へ (host1-2)
lsw2: 空
lsw3: 空
lsw4: 空
```
<br />

###### 実行結果
動作テストの実行結果を，以下に示す．  
まず，**host1-1**->**host1-2**へパケットを送信し，`show_stats`で各ホストが送受信したパケットの統計情報を確認する．  
また全てのスイッチのフローテーブルの確認も行った．  
結果，**host1-1**から**host1-2**へパケットが送信されていることが確認でき，各スイッチのフローテーブルは空であることが確認できた．

```
$ bin/trema send_packet --source "host1-1" --dest "host1-2"
$ bin/trema show_stats "host1-1"
Packets sent:
  192.168.0.11 -> 192.168.0.12 = 1 packet
$ bin/trema show_stats "host1-2"
Packets received:
  192.168.0.11 -> 192.168.0.12 = 1 packet
$ bin/trema dump_flows lsw1
NXST_FLOW reply (xid=0x4):
$ bin/trema dump_flows lsw2
NXST_FLOW reply (xid=0x4):
$ bin/trema dump_flows lsw3
NXST_FLOW reply (xid=0x4):
$ bin/trema dump_flows lsw4
NXST_FLOW reply (xid=0x4):
```  
<br />

続けて，**host1-2**->**host1-1**へパケットを送信し，`show_stats`で各ホストが送受信したパケットの統計情報を確認する．  
また全てのスイッチのフローテーブルの確認も行った．  
結果，**host1-2**から**host1-1**へパケットが送信されていることが確認できた．  
各スイッチのフローテーブルは前述したとおりになっていることが確認でき，フローテーブルの管理も含めて正常に動作しているといえる．(dl_src, dl_dst, actionsの値を確認すると，前述通りのフローテーブルになっていることが分かる)


```
$ bin/trema send_packet --source "host1-2" --dest "host1-1"
$ bin/trema show_stats "host1-1"
Packets sent:
  192.168.0.11 -> 192.168.0.12 = 1 packet
Packets received:
  192.168.0.12 -> 192.168.0.11 = 1 packet
$ bin/trema show_stats "host1-2"
Packets sent:
  192.168.0.12 -> 192.168.0.11 = 1 packet
Packets received:
  192.168.0.11 -> 192.168.0.12 = 1 packet
$ bin/trema dump_flows lsw1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=12.19s, table=0, n_packets=0, n_bytes=0, idle_age=12,
priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=00:00:00:00:00:12,dl_dst=00:00:00:00:00:11,
nw_src=192.168.0.12,nw_dst=192.168.0.11,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
$ bin/trema dump_flows lsw2
NXST_FLOW reply (xid=0x4):
$ bin/trema dump_flows lsw3
NXST_FLOW reply (xid=0x4):
$ bin/trema dump_flows lsw4
NXST_FLOW reply (xid=0x4):
```

そして再度，**host1-1**->**host1-2**へパケットを送信し，`show_stats`で各ホストが送受信したパケットの統計情報を確認する．  
また全てのスイッチのフローテーブルの確認も行った．  
結果，**host1-1**から**host1-2**へ再度パケットが送信されていることが確認できた．  
各スイッチのフローテーブルは前述したとおりになっていることが確認でき，フローテーブルの管理も含めて正常に動作しているといえる．(dl_src, dl_dst, actionsの値を確認すると，前述通りのフローテーブルになっていることが分かる)


```
$ bin/trema send_packet --source "host1-1" --dest "host1-2"
$ bin/trema show_stats "host1-1"
Packets sent:
  192.168.0.11 -> 192.168.0.12 = 2 packets
Packets received:
  192.168.0.12 -> 192.168.0.11 = 1 packet
$ bin/trema show_stats "host1-2"
Packets sent:
  192.168.0.12 -> 192.168.0.11 = 1 packet
Packets received:
  192.168.0.11 -> 192.168.0.12 = 2 packets
$ bin/trema dump_flows lsw1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=8.085s, table=0, n_packets=0, n_bytes=0, idle_age=8,
priority=65535,udp,in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:11,dl_dst=00:00:00:00:00:12,
nw_src=192.168.0.11,nw_dst=192.168.0.12,nw_tos=0,tp_src=0,tp_dst=0 actions=output:2
 cookie=0x0, duration=272.267s, table=0, n_packets=0, n_bytes=0, idle_age=272,
priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=00:00:00:00:00:12,dl_dst=00:00:00:00:00:11,
nw_src=192.168.0.12,nw_dst=192.168.0.11,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
$ bin/trema dump_flows lsw2
NXST_FLOW reply (xid=0x4):
$ bin/trema dump_flows lsw3
NXST_FLOW reply (xid=0x4):
$ bin/trema dump_flows lsw4
NXST_FLOW reply (xid=0x4):
```



#### 動作テスト その2
##### host1-1とhost2-1の送受信
###### 予想される動作
**host1-1**と**host2-1**の送受信のテストをした．  
**host1-1**から**host2-1**へパケットを送信する際に予想される動作を以下に述べる．

1. **host1-1**から**lsw1**へパケットが転送される
2. **lsw1**のフローテーブルが空なので，PacketInがコントローラに送られる． 
3. コントローラはPacketInメッセージから**host1-1**のMACアドレスとポート番号をFDB1に保存する．[FDB1; ***:11 => ポート1***]   
4. コントローラは送信先の**host2-1**のMACアドレスがFDB1に登録されていないので，パケットをフラッディングとしてPacketOutする．  
5. PacketOutを受け取った**lsw1**はパケットをフラッディングするので，結果，パケットが**host1-2**へ転送されるが，***host2-1***へは転送されない．
6. 結果，**host1-1**から**host2-1**へはパケットが伝送されない．

※処理後，全てのスイッチのフローテーブルは更新されていないので空のままのはずである．  


###### 実行結果
動作テストの実行結果を，以下に示す．  
まず，**host1-1**->**host2-1**へパケットを送信し，`show_stats`で各ホストが送受信したパケットの統計情報を確認する．  
また全てのスイッチのフローテーブルの確認も行った．  
結果，**host1-1**から**host2-1**へパケットが送信できておらず，各スイッチのフローテーブルは空であることが確認できた．なお，フラッディングで**host1-2**へパケットが転送されたが，宛先が異なり破棄されたので**host1-2**の`show_stats`は空であった．

```
$ bin/trema send_packet --source "host1-1" --dest "host2-1"
$ bin/trema show_stats "host1-1"
Packets sent:
  192.168.0.11 -> 192.168.0.21 = 1 packet
$ bin/trema show_stats "host1-2"
$ bin/trema show_stats "host2-1"
$ bin/trema dump_flows lsw1
NXST_FLOW reply (xid=0x4):
$ bin/trema dump_flows lsw2
NXST_FLOW reply (xid=0x4):
$ bin/trema dump_flows lsw3
NXST_FLOW reply (xid=0x4):
$ bin/trema dump_flows lsw4
NXST_FLOW reply (xid=0x4):
```  
