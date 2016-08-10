# 02_OS事前準備

**自分のリポジトリOpenStack on Ubuntu on オンプレ のコピッたまま。OpenStack on CentOS on ESXi 用に書き換え中。**


## OS準備


- CentOS 7 64bit のダウンロード [対象: 操作PC]

  - ブラウザにて、[理研からCentOS-7-x86_64-Minimal-1511.iso](http://ftp.riken.jp/Linux/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1511.iso) をダウンロード


- CentOS7 イメージの準備
  - vClientでESXiに接続し、OSイメージをアップロード

<img src="https://github.com/Soichiro75/openstack-liberty-on-centos-japanese/blob/master/02_OS事前準備/images/2016-08-09_010_OSアップロードtoESXi.png" width="320px" title="OSアップロードtoESXi.png">


## OSインストール

- 仮想マシン新規インストール

  - 以下を参考に controller01 と compute01 用のCentOS7をインストール
    - OpenStack用のVMとなるため、以下の表に記載の通り`Intel VT-x/AMD-Vを命令セット仮想化に使用し、ソフトウェアをMMU仮想化に使用`も有効にする必要がある
    - 補足:
      - [Intel VT](https://ja.wikipedia.org/wiki/インテル_バーチャライゼーション・テクノロジー)とは、仮想マシンモニタによる複数のOSの並行動作をより効率的に行うための支援技術
      - [MMU(Memory Management Unit)](https://ja.wikipedia.org/wiki/メモリ管理ユニット)とは、コンピュータのハードウェア部品のひとつであり、CPUの要求するメモリアクセスを処理する

|   |controller01用 VM|compute01用 VM|備考|
|---|---|---|---|
|構成|標準|標準|-|
|名前|controller01|compute01|-|
|ストレージ|datastore1|datastore1|-|
|ゲストOS|Linux|Linux|-|
|バージョン|Red Hat Enterprise Linux 7 (64ビット)|Red Hat Enterprise Linux 7 (64ビット)|ESXi5.5以下の場合は、「その他Linux(64bit)」|
|Nic|2つ|2つ|-|
|Nic 1|VM Network 101|VM Network 101|-|
|Nic 2|VM Network 102|VM Network 102|-|
|アダプタ(Nic1 & Nic2)|VMXNET 3|VMXNET 3|-|
|パワーオン時に接続(Nic1 & Nic2)|有効|有効|-|
|仮想ディスクサイズ|200GB|200GB|-|
|プロビジョニング|Thin Provision|Thin Provision|-|
|(確認)完了前に仮想マシンの設定を編集|有効|有効|-|
|以下、仮想マシンのプロパティ/ハードウェア|-|-|-|
|メモリ|24GB|24GB|-|
|CPU ソケット/コア|2/2|2/2|-|
|CD/DVD|データストアISO/CentOS7 Minimal|データストアISO/CentOS7 Minimal|-|
|CD/DVD パワーオン時に接続|有効|有効|-|
|以下、仮想マシンのプロパティ/オプション|-|-|-|
|CPU/MMU 仮想化|VT-xを使用~</br>ソフトウェアをMMU仮想化に使用|VT-xを使用~</br>ソフトウェアをMMU仮想化に使用|-|


- 以下を参考にCentOS7 をインストール


- controller01, compute01, cli01 とするサーバーに最小構成でインストールする
  - インストール時のパラメーター
    - Ubuntuインストール時に使用した言語がインストール後にも使用される。日本語環境では何かと問題が起きるので、英語環境とするために英語インストールを推奨
  - Block Storage のようなオプションサービスをインストールする場合には Logical Volume Manager (LVM) 推奨


|   |controller01用 VM|compute01用 VM|備考|
|---|---|---|---|
|Language ~ install process||||
|KEYBOAD|JAPANESE|JAPANESE|ENGLISH(US)は削除|
|DATE & TIME ZONE|Asia/Tokyo 現在日時を入力|Asia/Tokyo 現在日時を入力|Network Time は OFF のまま|
|LANGUAGE SUPPORT|ENGLISH(United States)|ENGLISH(United States)|デフォルト|
|SOFTWARE SELECTION|Minimal|Minimal Install|デフォルト|
|INSTALLATION DESTINATION|sda 200GB|sda 200GB||
|NETWORK 若番(VM Network 101)|-|-|設定画面に進むために`configure`押下|
|NETWORK 若番 Connection name|eth0|eth0|Connection nameはデフォルトの`enoxxxxxxxx` だと分かりにくいので`ethx`に書き換え|
|NETWORK 若番 IPv4 Method|Manual|Manual||
|NETWORK 若番 Address(IP/Netmask/DGW)|192.168.101.11 255.255.255.0 192.168.101.254|192.168.101.21 255.255.255.0 192.168.101.254||
|NETWORK 若番 DNS|8.8.8.8|8.8.8.8||
|NETWORK 若番 IPv6 Method|Ignore|Ignore|OpenStackでたぶん使わないと思う。。。。後で確認。|
|NETWORK 若番 General|Automatically connect~ にチェック|Automatically connect~ にチェック||
|NETWORK 老番(VM Network 102)|-|-|設定画面に進むために`configure`押下|
|NETWORK 老番 Connection name|eth1|eth1|Connection nameはデフォルトの`enoxxxxxxxx` だと分かりにくいので`ethx`に書き換え|
|NETWORK 老番 IPv6 Method|Ignore|Ignore|IPv4については設定しない(デフォルトのDHCPのまま)|
|NETWORK 老番 General|Automatically connect~ にチェック|Automatically connect~ にチェック||
|NETWORK 老番 ||||
|NETWORK 老番 ||||
|Host name|controller01|compute01||
|Bigin Installation|-|-||
|ROOT PASSWORD|Password123$|Password123$||
|Installing が完了するまで 約5分 待機|-|-||
|Reboot|-|-||


- (Option) vClientにて、controller01,compute01ともに、</br>
設定の編集 > CD/DVDドライブ1 > データストアISOファイル パワーオン時に接続 のチェックを外す > クライアントデバイス にチェック</br>
を実施しておく。</br>
ESXiにて、OVFテンプレート作成、デプロイ時にエラーになるのを防ぐため。


## ログイン確認

- ログイン確認
  - 操作PC のTera term から controller01(192.168.101.11) と compute01(192.168.101.21) に SSHログイン</br>
  ユーザー名/パスワード: root/Password123$


## ネットワーク設定

- 優先nicの設定確認
[対象: controller01, compute01, cli01]

```
# cat /etc/network/interfaces
========> controller01の例 cat ここから
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto enp3s0
iface enp3s0 inet static
        address 192.168.101.1
        netmask 255.255.255.0
        network 192.168.101.0
        broadcast 192.168.101.255
        gateway 192.168.101.254
        # dns-* options are implemented by the resolvconf package, if installed
        dns-nameservers 8.8.8.8
        dns-search st.local
========< cat ここまで

```

- プロバイダーインターフェースの設定
[対象: controller01, compute01]
  - `INTERFACE_NAME` は環境に合わせて置き換えること</br>
    ※ 私の環境ではcontroller01でh「enp6s0」、compute01では「enp2s0f1」
  - メモ: `iface <config_name> <address_family> <method_name>`
    - iface は スタンザと呼ばれる複数行からなる設定(他には、mapping, auto, source, allow-hotplug など)
    - method_name = static: 固定の設定を入れたい時に使用。address, netmask は必須。その他、gatewayなど。
    - method_name = manual: 自前で全部設定したい時に使用(今回の様に、address(IP)の設定なしでinterfaceの起動をしたい時などに使用)。 `/etc/network/if-\*.d/`以下のスクリプトで設定する。

```
# vi /etc/network/interfaces
========> 以下を追加 vi ここから
# The provider network interface
auto INTERFACE_NAME
iface INTERFACE_NAME inet manual
up ip link set dev $IFACE up
down ip link set dev $IFACE down
========< vi ここまで
```

- 名前解決
[対象: controller01, compute01, cli01]
  - (注意) ディストリビューションによっては`127.0.1.1`の記載が存在する、`127.0.1.1`が存在した場合は、不具合を防ぐため削除(コメントアウト)すること
  - `127.0.0.1` については削除しないこと

```
vi /etc/hosts
========> 以下を参考に編集 vi ここから
# 127.0.1.1     xxxx.st.local     xxxx
127.0.0.1       localhost
192.168.101.1   controller01.st.local     controller01
192.168.101.2   compute01.st.local     compute01
========< vi ここまで
```

- 接続確認
[対象: controller01, compute01, cli01]

  - hosts動作確認

```
# ping -c 4 controller01
# ping -c 4 controller01.st.local
# ping -c 4 compute01
# ping -c 4 compute01.st.local
```

  - インターネット接続確認

```
# ping -c 4 openstack.org
```

## NTP設定

controller01をNTPサーバーとして、その他のノードはcontroller01を参照して時刻同期する

### NTPサーバー設定

- Chrony インストール
[対象: controller01]

```
# apt-get install chrony
    Do you want to continue? [Y/n] Y
```

- Chrony 設定
[対象: controller01]
  - メモ: `iburst` オプション:サーバに到達できない場合、ポーリング間隔ごとに、通常のパケット1個の代わりに、パケット8個をバースト的に送信する。これによって、初期化時間を短縮出来る。

```
# vi /etc/chrony/chrony.conf
========> 以下を参考に編集 vi ここから
# pool 2.debian.pool.ntp.org offline iburst
server ntp.nict.jp iburst
allow 0/0
========< vi ここまで
```

- Chrony 設定反映
[対象: controller01]

```
# service chrony restart
```

- Chrony 同期確認
[対象: controller01]

```
# chronyc sources
設定したNTPサーバー(nict)に"^*"が付いていること
========> chronyc sources 結果 ここから
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* ntp-a2.nict.go.jp             1   6    17     3    +12us[ +121us] +/- 3852us
========< chronyc sources 結果 ここまで
```


### NTP クライアント設定

- Chrony インストール
[対象: compute01, cli01]

```
# apt-get install chrony
    Do you want to continue? [Y/n] Y
```

- Chrony 設定
[対象: compute01, cli01]

```
# vi /etc/chrony/chrony.conf
========> 以下を参考に編集 vi ここから
# pool 2.debian.pool.ntp.org offline iburst
server controller01 iburst
========< vi ここまで
```

- Chrony 設定反映
[対象: compute01, cli01]

```
# service chrony restart
```

- Chrony 同期確認
[対象: compute01, cli01]

```
# chronyc sources
controller01 に "^*" が付いていること
========> chronyc sources 結果 ここから
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* controller01.st.local         2   6    17     3   -763ns[  -13us] +/- 4261us
========< chronyc sources 結果 ここまで
```
