# 03_OS事前準備

**自分のリポジトリOpenStack on Ubuntu on オンプレ のコピッたまま。OpenStack on CentOS on ESXi 用に書き換え中。**

ネットワーク設定、DNS(今回はhosts)、NTP 等のOS周りの設定をする

## ネットワーク設定

- ログイン確認
  - 操作PC のTera term から controller01(192.168.101.11) と compute01(192.168.101.21) に SSHログイン</br>
  ユーザー名/パスワード: root/Password123$

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
