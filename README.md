# openstack-liberty-on-centos-japanese

***この手順は作成中です***


この手順では、
[OpenStack Docs Liberty Installation Guide for CentOS 7 ](http://docs.openstack.org/liberty/ja/install-guide-rdo/) を参考に、1Controller, 1Compute の構成でOpenStackのPoC環境を構築する

上記ディレクトリの各 README.md を順に行うことでPoC環境構築が出来るようになっている


注意書き：
- PoC環境なので、切り戻し等やり易い様にESXi上でOpenStackを構築する

- 各サーバー(OpenStackの各コンポーネント、今回はVM)がインターネットに繋がる環境であること

- 基本的にセキュリティは考慮しない

- 基本的にrootユーザーにて設定する

- パスワードは基本的に`Password123$`を使用する
  - [(参考資料)OpenStackで使用するパスワードについて](http://docs.openstack.org/liberty/ja/install-guide-rdo/environment-security.html)


自分用メモ:

パスワードは`Password123$`より、`PasswordAdmin`,`PasswordDemo`,`PasswordToken`みたいにした方が分かりやすかったかな。。。。。。


## 参考URL

- [OpenStack Docs Liberty Installation Guide for CentOS 7 ](http://docs.openstack.org/liberty/ja/install-guide-rdo/)
- [OpenStack構築手順書Liberty版 for Ubuntu(日本仮想化技術株式会社)](http://www.slideshare.net/VirtualTech-JP/openstackmitaka)
- [RDO Documentation](https://www.rdoproject.org/documentation/)
- [Documentation for Red Hat OpenStack Platform](https://access.redhat.com/documentation/ja/red-hat-openstack-platform?version=7/)


## 環境

### 使用機器(or VM)について

- 操作PC:

|   |Windows10|
|---|---|
|Application|Chrome, Tera_Term, </br>vSphere_Client(OpenStackをESXi上に構築するため)|


- 初期構築用 VM:

|   |controller01|compute01|
|---|---|---|
|CPU|4 cores|4 cores|
|Memory|24 GB|24 GB|
|HDD|200 GB|200 GB|
|Nic|2 nics|2 nics|


- スケールアウト用 VM:

|   |compute02|compute03|
|---|---|---|
|CPU|4 cores|4 cores|
|Memory|24 GB|24 GB|
|HDD|200 GB|200 GB|
|Nic|2 nics|2 nics|


### (参考)最小構成

[OpenStack ドキュメント 環境 について](http://docs.openstack.org/liberty/ja/install-guide-rdo/environment.html)

コアなサービスと CirrOS のインスタンスをいくつか動かす程度の検証環境(PoC)であれば、以下の最小要件で動作する

|   |controller01|compute01|
|---|---|---|
|CPU|1 core|1 core|
|Memory|4 GB|2 GB|
|HDD|5 GB|10 GB|
|Nic|1 nic|1 nic|


### (仮)構成図 (後できちんと描き直す)

<!-- <img src="https://github.com/Soichiro75/openstack-liberty-on-centos-japanese/blob/master/images/OpenStack構成図_超下書き.png" width="320px" title="OpenStack全体構成図">
-->

![OpenStack全体構成図 (超下書き)](https://github.com/Soichiro75/openstack-liberty-on-centos-japanese/blob/master/images/OpenStack構成図_超下書き.png)
