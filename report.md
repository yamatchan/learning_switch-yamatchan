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
vswitch('lsw1') {
  datapath_id 0xdef
}

vswitch('lsw2') {
  datapath_id 0xabc
}

vhost ('host1') {
  ip '192.168.0.1'
}

vhost ('host2') {
  ip '192.168.0.2'
}

vhost ('host3') {
  ip '192.168.0.3'
}

vhost ('host4') {
  ip '192.168.0.4'
}

link 'lsw1', 'host1'
link 'lsw1', 'host2'
link 'lsw2', 'host3'
link 'lsw2', 'host4'
```

#### 動作テスト
##### host1からhost2へ送信
**host1**から**host2**へパケットを送信した．  
**host1**と**host2**は**lsw1**に接続されているので，**lsw1**を介してパケットが転送されるはずである．   
動作テストの結果を，以下に示す．
`show_stats`で各ホストの送受信したパケットの統計情報を確認すると，正常に**host1**から**host2**へパケットが送信されていることが確認できる．

```
$ bin/trema packet_send --source host1 --dest host2
$ bin/trema show_stats host1
Packets sent:
  192.168.0.1 -> 192.168.0.2 = 1 packet
$ bin/trema show_stats host2
Packets received:
  192.168.0.1 -> 192.168.0.2 = 1 packet
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







