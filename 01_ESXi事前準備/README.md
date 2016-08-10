# 01_ESXi事前準備



## 事前確認

  - ESXiホストのCPUが`64bit`,`Virtualization Technology(VT-x or AMD-V)対応`であること
  - ESXiからインターネットに接続可能なネットワーク環境であること



## ESXi準備

- ESXiホストのBIOSにて`Virtualization Technology(VT-x or AMD-V)` を有効にしていること
- ESXiホストのBIOS時刻は、日本においてもUTCに設定してあること
  - [VMware ナレッジ](https://kb.vmware.com/selfservice/search.do?cmd=displayKC&docType=kc&docTypeID=DT_KB_1_1&externalId=2052726): ESXiはUTC以外のタイムゾーンをサポートしない。VMは起動時にESXiのBIOS時刻をUTCと解釈し参照する。その後、VM自体に設定されたタイムゾーンに合わせる動きをする。</br>
    - つまり、VMのタイムゾーンをJST かつ ESXiのBIOS時刻をJST時刻で設定していた場合、</br>
    VMはESXi_BIOS_JST時刻をUTC時刻と判断してしまうため、ESXi_BIOS_JST時刻からさらに +9時間足した時刻を起動時に持つ事になってしまう
- ESXi インストール

- IP 設定

- Hostname 設定

- 移行 vClient経由

- Network 二つ用意する

[イメージ図 挿入]()

- 両方のポートに、接続ポート の設定 無差別モード 承諾

[イメージ図 挿入]()

-
