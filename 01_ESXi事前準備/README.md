# 01_ESXi事前準備



## 事前確認

- ESXiホストのCPUが`64bit`,`Virtualization Technology(VT-x or AMD-V)対応`であること

- ESXiホストやVMからインターネットに接続可能なネットワーク環境であること

- ESXiのイメージをCD-ROMに焼いていること
  - 今回はESXi上にCentOS7を使用するため、[ESXi6.0](https://my.vmware.com/jp/group/vmware/evalcenter?p=free-esxi6)(ESXi5.5以上)を使用した



## ESXiインストール

- ESXiホストのBIOSにて`Virtualization Technology(VT-x or AMD-V)` を有効にする

- ESXiホストのBIOS時刻はUTC時刻で設定する(日本においてもUTC時刻で設定)
  - [VMware ナレッジ](https://kb.vmware.com/selfservice/search.do?cmd=displayKC&docType=kc&docTypeID=DT_KB_1_1&externalId=2052726): ESXiはUTC以外のタイムゾーンをサポートしない。VMは起動時にESXiの時刻(NTPの設定をしていないならBIOS時刻になる)をUTCと解釈し参照する。その後、VM自体に設定されたタイムゾーンに合わせる動きをする。</br>
    - つまり、VMのタイムゾーンをJST かつ ESXiのBIOS時刻をJST時刻 かつ ESXiにNTPの設定をしない場合、</br>
    VMはESXiの時刻(ESXi_BIOS_JST時刻)をUTC時刻と判断してしまうため、ESXi_BIOS_JST時刻からさらに +9時間足した時刻を起動時に持つ事になってしまう

- ESXi6.0 インストール

|設定項目|設定値|備考|
|---|---|---|
|Disk|適当なディスクを選択||
|キーボードレイアウト|Japanese||
|rootパスワード|Password123$||


## ESXi初期設定

- 再起動後、`F2`を押下し、インストール時に指定したrootパスワードでログイン

- `Configure Management Network`を選択

- `Network Adapters`で、一番若いNic番号`vmnic0`が選択されている事を確認

- `IPv4 Configuration`を選択

- IPv4 設定

|設定項目|設定値|備考|
|---|---|---|
|IPv4 Address|192.168.101.1||
|Subnet Mask|255.255.255.0||
|Default Gateway|192.168.101.254||

- `IPv6 Configuration`?を選択

|設定項目|設定値|備考|
|---|---|---|
|IPv6|無効||

- `DNS Configuration`を選択

- Hostname 設定

|設定項目|設定値|備考|
|---|---|---|
|Primary DNS Server|空白|
|Alternate DNS Server|空白|
|Hostname|esxi||


- 設定画面に戻り、`ESC`を押下、設定反映のため再起動を求められるので`Y`で再起動


## OpenStack on ESXi のための設定

### ネットワーク 作成

- 操作PCからvClientでログイン

- 必要であれば、datastoreの追加。今回はdatastore2を追加した。

- 以下を参考に `VM Network 101`と`VM Network 102`のネットワーク(ポートグループ)を作成

  - 192.168.101.1 ホスト選択 > 構成 > ハードウェア/ネットワーク > vSwitch0/プロパティ > ポート/VM Network 編集 >  全般/ネットワークラベル: VM Network 101

  - 192.168.101.1 ホスト選択 > 構成 > ハードウェア/ネットワーク > ネットワークの追加 > タイプ:仮想マシン > 標準スイッチの作成: vmnic1 > ネットワークラベル: VM Network 102

    - 上記`VM Network 102`を、vSwitch(と vmnic)を分けて作成しているが、vSwitch0に両ポートグループを作成しても構わない(と思う。 OpenStackのネットワークを詳しく理解した後に書き直すかも。)

<img src="https://github.com/Soichiro75/openstack-liberty-on-centos-japanese/blob/master/01_ESXi事前準備/images/2016-08-09_010_2PortGroups.png" width="320px" title="2PortGroups">

### Nested 設定

- 以下を参考に`VM Network 101`と`VM Network 102`両ポートグループに、`プロミスキャス(無差別)モード 承諾`(OpenStack上のVMと通信をとれるようにするため)を設定

  - 192.168.101.1 ホスト選択 > 構成 > ハードウェア/ネットワーク > vSwitch(0 or 1)/プロパティ > VM Network (101 or 102) 編集 > セキュリティ/無差別モード 承諾

<img src="https://github.com/Soichiro75/openstack-liberty-on-centos-japanese/blob/master/01_ESXi事前準備/images/2016-08-09_020_プロミスキャス(無差別)モード.png" width="320px" title="プロミスキャス(無差別)モード">
