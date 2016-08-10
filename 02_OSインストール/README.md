# 02_OSインストール

OpenStackの controller01 と compute01 用のVMとなるCentOS7 をインストールする


## OS準備


- CentOS 7 64bit のダウンロード [対象: 操作PC]

  - ブラウザにて、[理研からCentOS-7-x86_64-Minimal-1511.iso](http://ftp.riken.jp/Linux/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1511.iso) をダウンロード


- CentOS7 イメージの準備
  - vClientでESXiに接続し、OSイメージをアップロード

<img src="https://github.com/Soichiro75/openstack-liberty-on-centos-japanese/blob/master/02_OSインストール/images/2016-08-09_010_OSアップロードtoESXi.png" width="320px" title="OSアップロードtoESXi.png">


## OSインストール

- 新規仮想マシン作成

  - 以下を参考に controller01 と compute01 用のVM作成
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
  - Block Storage のようなオプションサービスをインストールする場合(今回はcontroler01にインストール)には [Logical Volume Manager (LVM) 推奨](http://docs.openstack.org/liberty/ja/install-guide-rdo/environment.html)らしいが、デフォルトでLVMなので以下インストール時に特に指定はしていない


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
