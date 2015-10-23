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


### 動作確認その1
#### 想定環境
スイッチを2つ(lsw1, lsw2)を用意し，それぞれのスイッチにホストを2つ接続し(lsw1; host1/host2,lsw2; host3/host4)，動作確認を行った．  
接続のイメージ図とconfファイルの詳細を以下に示す．

```
【イメージ図】
lsw1
┣ host1
┗ host2

lsw2
┣ host3
┗ host4
```
```
vswitch('lsw1') { datapath_id 0xdef }
vswitch('lsw2') { datapath_id 0xabc }

vhost ('host1') { ip '192.168.0.1' }
vhost ('host2') { ip '192.168.0.2' }
vhost ('host3') { ip '192.168.0.3' }
vhost ('host4') { ip '192.168.0.4' }

link 'lsw1', 'host1'
link 'lsw1', 'host2'
link 'lsw2', 'host3'
link 'lsw2', 'host4'
```

#### 動作テスト
##### host1とhost2の送受信
**host1**と**host2**のパケット送受信のテストをした．  
**host1**と**host2**は**lsw1**に接続されているので，お互いに**lsw1**を介してパケットが転送されるはずである．   
動作テストの結果を，以下に示す．  
まず，**host1->host2**へパケットを送信し，`show_stats`で各ホストが送受信したパケットの統計情報を確認する．  
そして，**host2->host1**へパケットを送信し，`show_stats`で各ホストが送受信したパケットの統計情報を確認した．  
結果，正常に**host1**,**host2**間でパケットが送受信されていることが確認できた．  
また，**lsw1**のフローテーブルを確認したところ，**host2->host1**のフローエントリのみが登録されており，正常に動作しているといえる．  

```
$ bin/trema packet_send --source host1 --dest host2
$ bin/trema show_stats host1
Packets sent:
  192.168.0.1 -> 192.168.0.2 = 1 packet
$ bin/trema show_stats host2
Packets received:
  192.168.0.1 -> 192.168.0.2 = 1 packet
$ bin/trema packet_send --source host2 --dest host1
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
$ bin/trema dump_flows lsw1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=204.758s, table=0, n_packets=0, n_bytes=0, idle_age=204,
priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=98:29:11:ca:df:bd,dl_dst=7b:52:dc:06:25:ec,
nw_src=192.168.0.2,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
```

##### host1からhost3へ送信
**host1**から**host3**へパケットを送信した．  
上述のイメージ図からも分かるように，**host1**は**lsw1**，**host3**は**lsw2**に接続されており，**lsw1**と**lsw2**は接続されていないので，
**host1**から**host3**へはパケットが転送されないはずである．   
動作テストの結果を，以下に示す．
`show_stats`で各ホストの送受信したパケットの統計情報を確認すると，**host1**からパケットは送信されているが，**host3**はそのパケットを受信できていないことが分かる．

```
$ bin/trema packet_send --source host1 --dest host3
$ bin/trema show_stats host1
Packets sent:
  192.168.0.1 -> 192.168.0.3 = 1 packet
$ bin/trema show_stats host3
```

##### host3からhost4へ送信
**host3**から**host4**へパケットを送信した．  
**host3**と**host4**は**lsw2**に接続されているので，**lsw2**を介してパケットが転送されるはずである．   
動作テストの結果を，以下に示す．
`show_stats`で各ホストの送受信したパケットの統計情報を確認すると，正常に**host3**から**host4**へパケットが送信されていることが確認できる．

```
$ bin/trema packet_send --source host3 --dest host4
$ bin/trema show_stats host3
Packets sent:
  192.168.0.3 -> 192.168.0.4 = 1 packet
$ bin/trema show_stats host4
Packets received:
  192.168.0.3 -> 192.168.0.4 = 1 packet
```




### 動作確認その2
#### 想定環境
スイッチを4つ(lsw1, lsw2, lsw3, lws4)を用意し，それぞれのスイッチにホストを1つ接続し(lsw1; host1, lsw2; host2, lsw3; host3, lsw4l host4)，さらにlsw1とlws2，lsw2とlws3，lws3とlws4を接続し動作確認を行った．  
接続のイメージ図とconfファイルの詳細を以下に示す．

<img src="https://github.com/handai-trema/learning_switch-yamatchan/blob/master/img/img1.png?raw=true">

```
vswitch('lsw1') { datapath_id 0x1 }
vswitch('lsw2') { datapath_id 0x2 }
vswitch('lsw3') { datapath_id 0x3 }
vswitch('lsw4') { datapath_id 0x4 }

vhost('host1') {
 ip '192.168.0.1'
 mac '00:00:00:00:00:01'
}
vhost('host2') {
 ip '192.168.0.2'
 mac '00:00:00:00:00:02'
}
vhost('host3') {
 ip '192.168.0.3'
 mac '00:00:00:00:00:03'
}
vhost('host4') {
 ip '192.168.0.4'
 mac '00:00:00:00:00:04'
}

link 'lsw1', 'host1'
link 'lsw2', 'host2'
link 'lsw3', 'host3'
link 'lsw4', 'host4'
link 'lsw1', 'lsw2'
link 'lsw2', 'lsw3'
link 'lsw3', 'lsw4'
```

#### 動作テスト
##### host1とhost2の送受信
**host1**と**host2**の送受信のテストをした．  
**host1**から**host2**へパケットを送信する際に予想される動作を以下に述べる．(MACアドレスはコロンを含めた末尾3桁に省略した形で記述する)  

1. **host1**から**lsw1**へパケットが転送される．  
2. **lsw1**のフローテーブルが空なので，PacketInがコントローラに送られる．  
3. コントローラはPacketInメッセージから**host1**のMACアドレスとポート番号をFDB1に保存する．[FDB1; :01 => ポート1]  
4. コントローラは送信先の**host2**のMACアドレスがFDB1に登録されていないので，パケットをフラッディングとしてPacketOutする．  
5. PacketOutを受け取った**lsw1**はパケットをフラッディングするので，結果，パケットが**lsw2**へ転送される．  
6. **lsw2**のフローテーブルも空なので，PacketInがコントローラに送られる．  
7. コントローラはPacketInメッセージから**host1**のMACアドレスとポート番号(**lsw1**の接続ポート)をFDB2に保存する．[FDB2; :01 => ポート2]   
8. コントローラは送信先の**host2**のMACアドレスがFDB2に登録されていないので，パケットをフラッディングとしてPacketOutする．  
9. PacketOutを受け取った**lsw2**はパケットをフラッディングするので，結果，パケットが**host2**および**lsw3**へ転送される(**lsw3**へ送られたパケットの処理は同様なので省略する)．  
10. **host2**が**lsw2**からパケットを受け取る．  

<img src="https://github.com/handai-trema/learning_switch-yamatchan/blob/master/img/img23.png?raw=true">

※処理後，全てのスイッチのフローテーブルは更新されていないので空のままのはずである．  
<br />

続けて，**host2**から**host1**へパケットを送信する際に予想される動作を以下に述べる．

11. **host2**から**lsw2**へパケットが転送される．
12. **lsw2**のフローテーブルが空なので，PacketInがコントローラに送られる．
13. コントローラはPacketInメッセージから**host2**のMACアドレスとポート番号をFDB2に保存する．[FDB2; :01 => ポート2, :02 => ポート1]  
さらに，送信先の**host1**のMACアドレスがFDB2に登録されているので，フローテーブルFT2を更新する．[FT2; :01->:02 => ポート2へ]  
14. コントローラは送信先の**host1**の属するネットワークのポート番号が分かっているので，送信ポート番号(ポート2)を付加してPacketOutする．
15. PacketOutを受け取った**lsw2**はパケットをポート2(**lsw1**)へパケットを転送する．
16. パケットを受け取った**lsw1**のフローテーブルも空なので，PacketInがコントローラに送られる．
17. コントローラはPacketInメッセージからhost2のMACアドレスとポート番号(lsw2の接続ポート)をFDB1に保存する．[FDB1; :01 => ポート1, :02 => ポート2]  
さらに，送信先の**host1**のMACアドレスがFDB1に登録されているので，フローテーブルFT1を更新する．[FT1; :01->:02 => ポート1へ]  
18. コントローラは送信先の**host1**のポート番号が分かっているので，送信ポート番号(ポート1)を付加してPacketOutする．
19. PacketOutを受け取った**lsw1**はパケットをポート1(**host1**)へパケットを転送する．
20. **host1**が**lsw1**からパケットを受け取る．  

<img src="https://github.com/handai-trema/learning_switch-yamatchan/blob/master/img/img45.png?raw=true">

処理後，それぞれのスイッチのフローテーブルは以下のようになっているはずである．

```
lsw1: :01->:02 => ポート1へ (host1)
lsw2: :01->:02 => ポート2へ (lsw1)
lsw3: 空
lsw4: 空
```
<br />

そして，動作テストの結果を，以下に示す．  
まず，**host1**->**host2**へパケットを送信し，`show_stats`で各ホストが送受信したパケットの統計情報を確認する．  
また**lsw1**と**lsw2**のフローテーブルの確認も行った．
結果，**host1**から**host2**へパケットが送信されていることが確認でき，各スイッチのフローテーブルは空であることが確認できた．

```
$ bin/trema send_packet --source host1 --dest host2
$ bin/trema show_stats host1
Packets sent:
  192.168.0.1 -> 192.168.0.2 = 1 packet
$ bin/trema show_stats host2
Packets received:
  192.168.0.1 -> 192.168.0.2 = 1 packet
$ bin/trema dump_flows lsw1
NXST_FLOW reply (xid=0x4):
$ bin/trema dump_flows lsw2
NXST_FLOW reply (xid=0x4):
```  
<br />

続けて，**host2**->**host1**へパケットを送信し，`show_stats`で各ホストが送受信したパケットの統計情報を確認する．  
また全てのスイッチのフローテーブルの確認も行った．  
結果，**host2**から**host1**へパケットが送信されていることが確認できた．
各スイッチのフローテーブルは前述したとおりになっていることが確認でき，フローテーブルの管理も含めて正常に動作しているといえる．


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
$ bin/trema dump_flows lsw1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=12.991s, table=0, n_packets=0, n_bytes=0, idle_age=12,
priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=00:00:00:00:00:02,dl_dst=00:00:00:00:00:01,
nw_src=192.168.0.2,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
$ bin/trema dump_flows lsw2
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=14.994s, table=0, n_packets=0, n_bytes=0, idle_age=14,
priority=65535,udp,in_port=1,vlan_tci=0x0000,dl_src=00:00:00:00:00:02,dl_dst=00:00:00:00:00:01,
nw_src=192.168.0.2,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 actions=output:2
$ bin/trema dump_flows lsw3
NXST_FLOW reply (xid=0x4):
$ bin/trema dump_flows lsw4
NXST_FLOW reply (xid=0x4):
```

#### 動作チェックの補足
`***`の項目を確認すると，きちんと予想通りのフローテーブルができていることが分かる．  

$ bin/trema dump_flows lsw1  
NXST_FLOW reply (xid=0x4):  
  cookie=0x0, duration=12.991s, table=0, n_packets=0, n_bytes=0, idle_age=12,  priority=65535,udp,in_port=2,vlan_tci=0x0000,`dl_src=00:00:00:00:00:02`,`dl_dst=00:00:00:00:00:01`,nw_src=192.168.0.2,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 `actions=output:1`

$ bin/trema dump_flows lsw2  
NXST_FLOW reply (xid=0x4):  
 cookie=0x0, duration=14.994s, table=0, n_packets=0, n_bytes=0, idle_age=14,  priority=65535,udp,in_port=1,vlan_tci=0x0000,`dl_src=00:00:00:00:00:02`,`dl_dst=00:00:00:00:00:01`,nw_src=192.168.0.2,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 `actions=output:2`
