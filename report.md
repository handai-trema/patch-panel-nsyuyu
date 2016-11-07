# 第三回レポート課題(10/19出題，10/25解答)
## 課題内容(パッチパネルの機能拡張)
パッチパネルに機能を追加せよ．

授業で説明したパッチの追加と削除以外に，以下の機能をパッチパネルに追加せよ．

1. ポートのミラーリング
2. パッチとポートミラーリングの一覧

それぞれ patch_panel のサブコマンドとして実装せよ．

なお 1 と 2 以外にも機能を追加した人には、ボーナス点を加点する．

## 解答
ポートのミラーリング機能，パッチとポートミラーリングの一覧表示機能の他に
パッチ追加時のエラー処理を実装した．

### ポートのミラーリング機能について

#### 仕様
ミラーリング機能を実行するコマンドの仕様を以下に示す．
```
./bin/patch_panel create データパスid 監視対象となるポート番号(モニターポート) ミラーポート番号

# 例 (ポート1を監視し，パケットの内容をポート3へミラーしたい場合)
./bin/patch_panel create 0xabc 1 3
```

#### 実装
ポートのミラーリング機能の実現にあたって，port_mirroring関数と，
add_mirror_flow_entries関数，patch_panelプロセスからport_mirroring関数を
呼び出す機能を実装した．
また，ミラーポートを保存する変数@mirrorを追加した．
元のソースコードのstart関数に不適切な記述があり，修正を行ったが
それについては，後述の"元のソースコードの不適切な記述について"の項目で述べる．
port_mirroring関数のソースコードを以下に示す．
```ruby
 40 def port_mirroring(dpid, port_monitor, port_mirror)
 41   port_monitor_a = port_monitor
 42   port_monitor_b = -1
 43   @patch[dpid].each do |port_a, port_b|
 44     if port_a == port_monitor_a then
 45       port_monitor_b = port_b
 46       break
 47     elsif port_b == port_monitor_a then
 48       port_monitor_b = port_a
 49       break
 50     end
 51   end
 52   if port_monitor_b == -1 then
 53     puts "error: not exist patch"
 54   else
 55     add_mirror_flow_entries dpid, port_monitor_a, port_monitor_b, port_mirror
 56     @mirror[dpid].push([port_monitor,port_mirror])
 57   end
 58 end
```
port_mirroring関数は，patch_panelプロセスから呼び出される．
port_mirroring関数では，まず，監視されるポート(モニターポート)が，パッチに接続されているか
どうかを調べ，監視対象となるポートに接続されているパッチのもう一方のポートを検索する(43行目〜51行目)．
パッチに接続されていなければ，そのポートは，パケットの送受信を行うことができず，
監視の対象として，不適切であるので，エラーメッセージを出力する(53行目)．
監視対象となるポートに接続されているパッチのもう一方のポートを検索する理由は，監視対象となるポートに
対して送信されるパケットをミラーポートへ出力するためには，送信元となるポートを知る必要があるからである．
監視対象となるポートがあるパッチに接続されていれば，add_mirror_flow_entires関数によって，ミラーポートへの出力を
行うフローエントリを追加する(55行目)．
そして，ミラーポートの一覧を保存する変数に，今回のミラーリングの設定を追加する．

次に，add_mirror_flow_entries関数のソースコードを以下に示す．
```ruby
 88 def add_mirror_flow_entries(dpid, port_monitor_a, port_monitor_b, port_mirror)
 89   send_flow_mod_delete(dpid, match: Match.new(in_port: port_monitor_a))
 90   send_flow_mod_delete(dpid, match: Match.new(in_port: port_monitor_b))
 91   send_flow_mod_add(dpid,
 92                    match: Match.new(in_port: port_monitor_a),
 93                    actions: [
 94                              SendOutPort.new(port_monitor_b),
 95                              SendOutPort.new(port_mirror),
 96                             ])
 97   send_flow_mod_add(dpid,
 98                    match: Match.new(in_port: port_monitor_b),
 99                    actions: [
100                               SendOutPort.new(port_monitor_a),
101                               SendOutPort.new(port_mirror),
102                              ])
103 end
```
add_mirror_flow_entries関数では，ミラーポートへの出力に関するフローエントリを追加する．
まず，監視対象となるポートと，監視対象となるポートとパッチで接続されているポートに関するフローエントリを
削除する(89〜90行目)．
そして，アクションに，ミラーポートへのパケットの出力を追加したフローエントリをフローテーブルに追加する(91〜102行目)．

最後に，patch_panelプロセスのプログラムの追記部分を以下に示す．
```
 44 desc 'Mirror'
 45   arg_name 'dpid port#1 port#2'
 46   command :mirror do |c|
 47     c.desc 'Location to find socket files'
 48     c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR
 49
 50     c.action do|_global_options, options, args|
 51       dpid = args[0].hex
 52       port1 = args[1].to_i
 53       port2 = args[2].to_i
 54       Trema.trema_process('PatchPanel', options[:socket_dir]).controller.
 55         port_mirroring(dpid, port1, port2)
 56     end
 57 end
```
54〜56行目の部分で，port_mirroring関数を呼び出す設定を行っている．

#### 動作確認
動作確認の際に使用した設定ファイルを以下に示す．
```
vswitch('patch_panel') { datapath_id 0xabc }

vhost ('host1') { ip '192.168.0.1' }
vhost ('host2') { ip '192.168.0.2' }
vhost ('host3') {
  ip '192.168.0.3'
  promisc true
}

link 'patch_panel', 'host1'
link 'patch_panel', 'host2'
link 'patch_panel', 'host3'
```
ホストの数は，3つとし，host3は宛先が自分以外のパケットの受信が行えるように，
promiscオプションの値をtrueに設定した．

動作確認は以下の手順で行った．

1. port1とport2を接続するパッチを作成する
2. ミラーポートを設定する(モニターポートはport1，ミラーポートはport3)
3. port1からport2へパケットを送信する．
4. port2からport1へパケットを送信する．
5. host1，host2，host3のパケット送受信情報を表示する．
6. フローテーブルを表示する．

実行結果を以下に示す．
```
$ ./bin/patch_panel create 0xabc 1 2
$ ./bin/patch_panel mirror 0xabc 1 3
$ ./bin/trema send_packets --source host1 --dest host2
$ ./bin/trema send_packets --source host2 --dest host1
$ ./bin/trema show_stats host1
Packets sent:
  192.168.0.1 -> 192.168.0.2 = 1 packet
Packets received:
  192.168.0.2 -> 192.168.0.1 = 1 packet
$ ./bin/trema show_stats host2
Packets sent:
  192.168.0.2 -> 192.168.0.1 = 1 packet
Packets received:
  192.168.0.1 -> 192.168.0.2 = 1 packet
$ ./bin/trema show_stats host3
Packets received:
  192.168.0.1 -> 192.168.0.2 = 1 packet
  192.168.0.2 -> 192.168.0.1 = 1 packet
$ ./bin/trema dump_flows patch_panel
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=77.703s, table=0, n_packets=1, n_bytes=42, idle_age=39, priority=0,in_port=1 actions=output:2,output:3
 cookie=0x0, duration=77.693s, table=0, n_packets=1, n_bytes=42, idle_age=28, priority=0,in_port=2 actions=output:1,output:3
```
手順5のhost3のパケット送受信情報の表示の結果において，port1からport2，port2からport1へ送信されたパケットを
port3に接続されたhost3が受信していることがわかり，モニターポートであるport1を介して送受信されるパケットを，
ミラーポートであるport3に送出できていることがわかる．
また，フローテーブルのactionsを見ると，port1，2から送信されるパケットはport3へ出力するようなエントリが
追加されていることがわかり，ミラーリング機能を正しく実装できたことが確認できる．

### パッチとポートミラーリングの一覧表示機能

#### 仕様
パッチとポートミラーリングの一覧表示機能を実行するコマンドの仕様を以下に示す．
```
./bin/patch_panel list データパスid
# 以下は出力結果の仕様
---------- patch list ----------
* port番号 <-> port番号
* port番号 <-> port番号
...

---------- mirror list ---------
* monitor: モニターポート番号 --> mirror: ミラーポート番号
...

# 例
./bin/patch_panel list 0xabc
# 以下は出力結果の例
---------- patch list ----------
* 1 <-> 2

---------- mirror list ---------
* monitor: 1 --> mirror: 3
...

```

#### 実装
パッチとポートミラーリングの一覧表示機能の実現にあたって，show_patch_mirror_list関数と，
patch_panelプロセスからshow_patch_mirror_list関数を呼び出す機能を実装した．
show_patch_mirror_list関数のソースコードを以下に示す．
```ruby
60 def show_patch_mirror_list(dpid)
61   puts "---------- patch list ----------"
62   @patch[dpid].each do |port_a, port_b|
63     puts "* #{port_a} <-> #{port_b}"
64   end
65   puts ""
66   puts "---------- mirror list ---------"
67   @mirror[dpid].each do |port_a, port_b|
68     puts "* monitor: #{port_a} --> mirror: #{port_b}"
69   end
70 end
```
まず，@patch変数からパッチの一覧を取り出し，仕様のフォーマットに従って表示する(61〜64行目)．
そして，@mirror変数からミラーリングの設定の一覧を取り出し，仕様のフォーマットに従って表示する(66〜69行目)

patch_panelプロセスのプログラムの追記部分を以下に示す．
```
59 desc 'List'
60 arg_name 'dpid'
61 command :list do |c|
62  c.desc 'Location to find socket files'
63  c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR
64
65  c.action do|_global_options, options, args|
66    dpid = args[0].hex
67    Trema.trema_process('PatchPanel', options[:socket_dir]).controller.
68      show_patch_mirror_list(dpid)
69  end
70 end
```
67〜68行目の部分で，port_mirroring関数を呼び出す設定を行っている．

#### 動作確認
動作確認の際に使用した設定ファイルはポートのミラーリング機能の動作確認で使用した
ファイルと同じであるので，詳細は省略する．

動作確認は以下の手順で行った．

1. port1とport2を接続するパッチを作成する
2. ミラーポートを設定する(モニターポートはport1，ミラーポートはport3)
3. listコマンドを実行する．

実行結果を以下に示す．
```
$ ./bin/patch_panel create 0xabc 1 2
$ ./bin/patch_panel mirror 0xabc 1 3
$ ./bin/patch_panel list 0xabc
# 以下はtrema runプロセスの出力結果
---------- patch list ----------
* 1 <-> 2

---------- mirror list ---------
* monitor: 1 --> mirror: 3
```
上記の結果からパッチとポートの一覧を正しく表示でき，一覧表示機能を正しく実装できたことが
確認できた．

### パッチ追加時のエラー処理
パッチを追加する際，対象とするポートに既にパッチが存在する場合は，パッチを追加することはできない．
従来のソースコードでは，この操作に対するエラー処理がなされておらず，create_patch関数内にエラー処理を実装した．
エラー処理実装後のcreate_patch関数のソースコードを以下に示す．
```ruby
17 def create_patch(dpid, port_a, port_b)
18   already_flag = 0
19   @patch[dpid].each do |port1, port2|
20     if port1 == port_a || port2 == port_a ||
21         port1 == port_b || port2 == port_b
22       already_flag = 1
23       break
24     end
25   end
26   if already_flag == 0 then
27     add_flow_entries dpid, port_a, port_b
28     @patch[dpid].push([port_a, port_b].sort)
29   else
30     puts "error: this port already has patch"
31   end
32 end
```
まず，パッチの一覧を保存している，@patch変数の中身を参照し，対象となるポートに既にパッチが存在しているかどうかを
調べ，存在している場合はフラグ用の変数(already_flag)の値を1にする(19行目〜25行目)．
already_flagの値が0，つまり，対象となるポートに対するパッチが存在してない場合，
フローエントリを追加し，パッチを作成する(26〜28行目)．
already_flagの値が1，つまり，対象となるポートに既にパッチが存在している場合，
エラーメッセージを表示する(29〜31行目)．

#### 動作確認
動作確認の際に使用した設定ファイルはポートのミラーリング機能の動作確認で使用した
ファイルと同じであるので，詳細は省略する．

動作確認は以下の手順で行った．

1. port1とport2を接続するパッチを作成する
2. port1とport3を接続するパッチを作成する
3. port2とport3を接続するパッチを作成する

実行結果を以下に示す．
```
$ ./bin/patch_panel create 0xabc 1 2
$ ./bin/patch_panel create 0xabc 1 3
error: this port already has patch #この行はtrema runプロセスの出力結果
$ ./bin/patch_panel create 0xabc 2 3
error: this port already has patch #この行はtrema runプロセスの出力結果
```
上記の結果からパッチ追加時のエラー処理を正しく実装できたことが
確認できた．

### 元のソースコードの不適切な記述について
スライドのプログラムの説明では，パッチのポート番号は配列で保存しているという旨のことが
書かれていたが，課題の元のソースコードでは，ハッシュの初期化を以下のように行っていた．
```ruby
def start(_args)
  @patch = Hash.new {[]}
  @mirror = Hash.new {[]}
  logger.info 'PatchPanel started.'
end
```
上記のプログラムでは，ハッシュの要素として，配列を持つことはできない．
よって，ハッシュの初期化の部分を以下のように修正した．
```ruby
def start(_args)
  @patch = Hash.new { |h,k| h[k] = [] }
  @mirror = Hash.new { |h,k| h[k] = [] }
  logger.info 'PatchPanel started.'
end
```
