# OS事前準備

ネットワーク設定、DNS(今回はhosts)、NTP、yumリボジトリ、OpenStackのパッケージインストール 等を設定する


## Network周りの設定

### 接続

- ログイン
  - 操作PC のTera term から controller01(192.168.101.11) と compute01(192.168.101.21) に SSHログイン</br>
  ユーザー名/パスワード: root/Password123$



### NetworkManagerサービス の無効化 と Networkサービス の有効化

- NetworkManagerサービスの無効化 [対象: controller01, compute01]

```
# systemctl disable NetworkManager
========>
Removed symlink /etc/systemd/system/multi-user.target.wants/NetworkManager.service.
Removed symlink /etc/systemd/system/dbus-org.freedesktop.NetworkManager.service.
Removed symlink /etc/systemd/system/dbus-org.freedesktop.nm-dispatcher.service.
========<

# systemctl stop NetworkManager
```

- Networkサービス の有効化 [対象: controller01, compute01]
```
# systemctl enable network
========>
network.service is not a native service, redirecting to /sbin/chkconfig.
Executing /sbin/chkconfig network on
========<

# chkconfig --list
========>
(省略)
network         0:off   1:off   2:on    3:on    4:on    5:on    6:off
========<

# systemctl start network
```


### デバイス名の確認

- デバイス名の確認 [対象: controller01, compute01]

```
# <環境によってデバイス名は異なる。(ここでは、eno16780032 と eno33559296)以降デバイス名については読み替えて実施すること>
# ip a
========>
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eno16780032: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 00:0c:29:f2:dc:1b brd ff:ff:ff:ff:ff:ff
    inet 192.168.101.11/24 brd 192.168.101.255 scope global eno16780032
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fef2:dc1b/64 scope link
       valid_lft forever preferred_lft forever
3: eno33559296: <BROADCAST,MULTICAST> mtu 1500 qdisc mq state DOWN qlen 1000
    link/ether 00:0c:29:f2:dc:25 brd ff:ff:ff:ff:ff:ff
========<
```


### ネットワークインターフェース の設定確認

- 確認 [対象: controller01, compute01]

```
# <管理インターフェース用>
# cat /etc/sysconfig/network-scripts/ifcfg-eno16780032
========>
TYPE=Ethernet
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
NAME=eno16780032
UUID=88c111a1-1101-4e78-ad31-b110566fbcb5
DEVICE=eno16780032
ONBOOT=yes
IPADDR=192.168.101.11
PREFIX=24
GATEWAY=192.168.101.254
DNS1=8.8.8.8
========<


# <パブリックインターフェース用>
# cat /etc/sysconfig/network-scripts/ifcfg-eno33559296
========>
[root@controller01 ~]# cat /etc/sysconfig/network-scripts/ifcfg-eno33559296
TYPE=Ethernet
BOOTPROTO=dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
NAME=eno33559296
UUID=ab9e9c54-a4df-4e00-b2ea-c3e1ca969c77
DEVICE=eno33559296
ONBOOT=yes
PEERDNS=yes
PEERROUTES=yes
========<
```

### パブリックインターフェース の設定

- 設定 [対象: controller01, compute01]

```
# vi /etc/sysconfig/network-scripts/ifcfg-eno33559296
========> 以下の部分のみ編集 他はそのまま
DEVICE=eno33559296
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
========<

# cat /etc/sysconfig/network-scripts/ifcfg-eno33559296
========>
TYPE=Ethernet
#BOOTPROTO=dhcp
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
NAME=eno33559296
UUID=ab9e9c54-a4df-4e00-b2ea-c3e1ca969c77
DEVICE=eno33559296
ONBOOT=yes
PEERDNS=yes
PEERROUTES=yes
========<
```


### 設定の反映

- 設定 [対象: controller01, compute01]

```
# reboot
```


## 名前解決

### hostsの設定 (DNSの変わり)

- hostsの設定 [対象: controller01, compute01]
[対象: controller01, compute01, cli01]
  - (注意) ディストリビューションによっては`127.0.1.1`の記載が存在する、`127.0.1.1`が存在した場合は、不具合を防ぐため削除(コメントアウト)すること
  - `127.0.0.1` については削除しないこと

```
vi /etc/hosts
========> 以下を追加
192.168.101.11   controller01
192.168.101.21   compute01
========<
```

- 接続確認

  - hosts動作確認 [対象: controller01, compute01]

```
# <疎通できること>
# ping -c 4 controller01
# ping -c 4 compute01
```

- インターネット接続確認 [対象: controller01, compute01]

```
# <疎通できること>
# ping -c 4 openstack.org
```

## ファイアウォールの無効化

- 無効化確認 [対象: controller01, compute01]

```
# systemctl status firewalld
========>
● firewalld.service
   Loaded: not-found (Reason: No such file or directory)
   Active: inactive (dead)
========<


# <firewalldがloadされている かつ 動作している場合は以下を実施し停止する>
# systemctl disable firewalld
# systemctl stop firewalld

```


## open-vm-tools インストール

https://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=2074713

CentOS7 ではVMware-toolsではなく、open-vm-toolsインストール推奨


- open-vm-tools インストール [対象: controller01, compute01]

```
# yum install -y open-vm-tools
```


- サービス有効化 [対象: controller01, compute01]

```
# service vmtoolsd restart
========>
Redirecting to /bin/systemctl restart  vmtoolsd.service
========<
```

- バージョン確認 [対象: controller01, compute01]

```
# vmtoolsd -v
========>
VMware Tools daemon, version 9.10.2.48224 (build-2822639)
========<
```

<!---
- キーのインポート [対象: controller01, compute01]

```
# rpm --import http://packages.vmware.com/tools/keys/VMWARE-PACKAGING-GPG-RSA-KEY.pub
```


- リポジトリ追加 [対象: controller01, compute01]

```
# vi /etc/yum.repos.d/vmware-tools.repo
[vmware-tools]
name = VMware Tools
baseurl = http://packages.vmware.com/packages/rhel7/x86_64/
enabled = 1
gpgcheck = 1
```


- パッケージインストール [対象: controller01, compute01]
```
# yum clean all
# yum list
# yum update
# yum install open-vm-tools-deploypkg

としたいが、以下エラー発生、そのため、open-vm-tools-deploypkgではなく、open-vm-toolsをインストールすることとする
->Finished Dependency Resolution
Error: Package: open-vm-tools-deploypkg-9.4.10-3.x86_64 (vmware-tools)
           Requires: open-vm-tools < 9.5
           Available: open-vm-tools-9.10.2-4.el7.x86_64 (base)
               open-vm-tools = 9.10.2-4.el7
           Available: open-vm-tools-9.10.2-5.el7_2.x86_64 (updates)
               open-vm-tools = 9.10.2-5.el7_2
Error: Package: open-vm-tools-deploypkg-9.4.10-3.x86_64 (vmware-tools)
           Requires: open-vm-tools < 9.5
           Available: open-vm-tools-9.10.2-4.el7.x86_64 (base)
               open-vm-tools = 9.10.2-4.el7
           Installing: open-vm-tools-9.10.2-5.el7_2.x86_64 (updates)
               open-vm-tools = 9.10.2-5.el7_2
 You could try using --skip-broken to work around the problem
 You could try running: rpm -Va --nofiles --nodigest

```

- vmtools再起動 [対象: controller01, compute01]

```
# service vmtoolsd restart
vmtoolsd を停止中:                                         [  OK  ]
vmtoolsd を起動中:                                         [  OK  ]
```


- vmtools 確認 [対象: controller01, compute01]

```
# vmtoolsd -v
VMware Tools daemon, version 9.4.6.33107 (build-1770165)
```

--->



## その他確認

- 確認 [対象: controller01, compute01]

```
# <正しく設定されていること>
# hostname

# <LANG=en_US.UTF-8 であること>
# localectl status
========>
System Locale: LANG=en_US.UTF-8
    VC Keymap: jp
   X11 Layout: jp
========<

# <Time zone: Asia/Tokyo であること>
# timedatectl status
========>
Local time: Wed 2016-08-10 17:24:41 JST
Universal time: Wed 2016-08-10 08:24:41 UTC
  RTC time: Wed 2016-08-10 08:24:42
 Time zone: Asia/Tokyo (JST, +0900)
NTP enabled: n/a
NTP synchronized: no
RTC in local TZ: no
DST active: n/a
========<

```


## NTP設定

controller01をNTPサーバーとして、その他のノードはcontroller01を参照して時刻同期する

### NTPサーバー設定

- Chrony インストール 設定 [対象: controller01 のみ]

```
# yum -y install chrony

# rpm -qa | grep chrony
========>
chrony-2.1.1-1.el7.centos.x86_64
========<

# <全セグメントから時刻同期許可>
# vi /etc/chrony.conf
========> 以下を参考に コメントアウト & 追加
# server 0.centos.pool.ntp.org iburst
# server 1.centos.pool.ntp.org iburst
# server 2.centos.pool.ntp.org iburst
# server 3.centos.pool.ntp.org iburst

server ntp.nict.jp iburst
allow 0/0
========<



# <chronyを使うので、ntpd停止の確認>
# systemctl status ntpd.service
========>
● ntpd.service
   Loaded: not-found (Reason: No such file or directory)
   Active: inactive (dead)
========<

# systemctl enable chronyd.service

# systemctl restart chronyd.service

# systemctl list-unit-files --type service | egrep "(ntp|chronyd)"
========>
chronyd.service   enabled
========<

# chronyc sources
========>
210 Number of sources = 1
MS Name/IP address    Stratum Poll Reach LastRx Last sample
====================================================================
^* ntp-b2.nict.go.jp  1   6    17    29   -260us[ -394us] +/- 4709us
========<
```


### NTPクライアント設定

- Chrony インストール 設定 [対象: compute01]

  - 補足: `iburst` オプション:サーバに到達できない場合、ポーリング間隔ごとに、通常のパケット1個の代わりに、パケット8個をバースト的に送信する。これによって、初期化時間を短縮出来る。


```
# yum -y install chrony

# rpm -qa | grep chrony
========>
chrony-2.1.1-1.el7.centos.x86_64
========<

# vi /etc/chrony.conf
========>
# <以下コメントアウト>
# server 0.centos.pool.ntp.org iburst
# server 1.centos.pool.ntp.org iburst
# server 2.centos.pool.ntp.org iburst
# server 3.centos.pool.ntp.org iburst

# <以下追加>
server controller01 iburst
========<

# <chronyを使うので、ntpd停止の確認>
# systemctl status ntpd.service
========>
● ntpd.service
   Loaded: not-found (Reason: No such file or directory)
   Active: inactive (dead)
========<

# systemctl enable chronyd.service

# systemctl restart chronyd.service

# systemctl list-unit-files --type service | egrep "(ntp|chronyd)"
========>
chronyd.service   enabled
========<

# chronyc sources
========>
210 Number of sources = 1
MS Name/IP address   Stratum Poll Reach LastRx Last sample
==========================================================
^* controller01      3       6    17    3  +2211ns[ +88us] +/- 13ms
========<
```


### OpenStack用リポジトリ

http://docs.openstack.org/liberty/ja/install-guide-rdo/environment-packages.html



## EPELの無効確認

- EPEL無効の確認 [対象: controller01, compute01]

EPELのいくつかのアップデートには後方互換性がないものがあるため、RDOパッケージを使用する場合には、EPELを無効にすることを推奨する

```
# <何も表示されないこと>
ls -l /etc/yum.repos.d/ | grep -i epel
```


## リポジトリ確認

CentOSの場合は、デフォルトで含まれるExtrasにRDO関連のパッケージが含まれているが、ReadHatの場合は別途追加が必要


- 確認 [対象: controller01, compute01]

```
grep -i Extras /etc/yum.repos.d/*
========>
/etc/yum.repos.d/CentOS-Base.repo:[extras]
/etc/yum.repos.d/CentOS-Base.repo:name=CentOS-$releasever - Extras
/etc/yum.repos.d/CentOS-Base.repo:mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras&infra=$infra
/etc/yum.repos.d/CentOS-Base.repo:#baseurl=http://mirror.centos.org/centos/$releasever/extras/$basearch/
/etc/yum.repos.d/CentOS-Sources.repo:[extras-source]
/etc/yum.repos.d/CentOS-Sources.repo:name=CentOS-$releasever - Extras Sources
/etc/yum.repos.d/CentOS-Sources.repo:baseurl=http://vault.centos.org/centos/$releasever/extras/Source/
/etc/yum.repos.d/CentOS-Vault.repo:[C7.0.1406-extras]
/etc/yum.repos.d/CentOS-Vault.repo:name=CentOS-7.0.1406 - Extras
/etc/yum.repos.d/CentOS-Vault.repo:baseurl=http://vault.centos.org/7.0.1406/extras/$basearch/
/etc/yum.repos.d/CentOS-Vault.repo:[C7.1.1503-extras]
/etc/yum.repos.d/CentOS-Vault.repo:name=CentOS-7.1.1503 - Extras
/etc/yum.repos.d/CentOS-Vault.repo:baseurl=http://vault.centos.org/7.1.1503/extras/$basearch/
```


  - (補足) RedHatの場合は実施する(この手順ではCentOSを使用のため実施しない 以下参考のため記載)

```
# subscription-manager repos --enable=rhel-7-server-optional-rpms
# subscription-manager repos --enable=rhel-7-server-extras-rpms
```


## OSパッケージの最新化

- OSパッケージのアップデート [対象: controller01, compute01]

```
# yum clean allow

# yum list

# <約10分弱>
# yum -y update

# reboot
```


## 自動アップデートの無効確認

OpenStack 環境に影響を与える可能性があるため、自動更新サービスは無効にする

- 無効確認 [対象: controller01, compute01]

```
# systemctl status yum-cron
========>
● yum-cron.service
   Loaded: not-found (Reason: No such file or directory)
   Active: inactive (dead)
========<


# cat /etc/yum/yum-cron.conf | grep apply_updates
========>
apply_updates = no
========<
```


## OpenStackリポジトリーの有効化(インストール)

- OpenStackのパッケージインストール [対象: controller01, compute01]

```
yum install -y centos-release-openstack-liberty
========>
(省略)
Installed:
  centos-release-openstack-liberty.noarch 0:1-4.el7

Complete!
========<
```

- (補足) RedHatの場合は上記の代わりに以下を実施する(この手順ではCentOSを使用のため実施しない 以下参考のため記載)

```
# yum install https://rdoproject.org/repos/openstack-liberty/rdo-release-liberty.rpm
```

## OpenStackクライアントのインストール [対象: controller01, compute01]

- クライアントのインストール [対象: controller01, compute01]

```
yum install -y python-openstackclient
 ========>
 Installed:
   python-openstackclient.noarch 0:1.7.2-1.el7

 Dependency Installed:
   PyYAML.x86_64 0:3.10-11.el7                                      libyaml.x86_64 0:0.1.4-11.el7_0
   pyOpenSSL.noarch 0:0.15.1-1.el7                                  pyparsing.noarch 0:2.0.3-1.el7
   python-backports.x86_64 0:1.0-8.el7                              python-backports-ssl_match_hostname.noarch 0:3.4.0.2-4.el7
   python-chardet.noarch 0:2.2.1-1.el7_1                            python-cinderclient.noarch 0:1.4.0-1.el7
   python-cliff.noarch 0:1.15.0-1.el7                               python-cliff-tablib.noarch 0:1.1-3.el7
   python-cmd2.noarch 0:0.6.8-3.el7                                 python-crypto.x86_64 0:2.6.1-1.el7.centos
   python-enum34.noarch 0:1.0.4-1.el7                               python-extras.noarch 0:0.0.3-2.el7
   python-fixtures.noarch 0:1.4.0-2.el7                             python-glanceclient.noarch 1:1.1.0-1.el7
   python-httplib2.noarch 0:0.9.2-1.el7                             python-idna.noarch 0:2.0-1.el7
   python-ipaddress.noarch 0:1.0.7-4.el7                            python-jsonpatch.noarch 0:1.2-3.el7.centos
   python-jsonpointer.noarch 0:1.9-2.el7                            python-jsonschema.noarch 0:2.3.0-1.el7
   python-keyring.noarch 0:5.0-4.el7                                python-keystoneclient.noarch 1:1.7.2-1.el7
   python-linecache2.noarch 0:1.0.0-1.el7                           python-mimeparse.noarch 0:0.1.4-1.el7
   python-monotonic.noarch 0:0.3-1.el7                              python-msgpack.x86_64 0:0.4.6-3.el7
   python-netaddr.noarch 0:0.7.18-1.el7                             python-netifaces.x86_64 0:0.10.4-1.el7
   python-neutronclient.noarch 0:3.1.0-1.el7                        python-novaclient.noarch 1:2.30.1-1.el7
   python-pbr.noarch 0:1.8.1-2.el7                                  python-ply.noarch 0:3.4-10.el7
   python-prettytable.noarch 0:0.7.2-2.el7.centos                   python-pycparser.noarch 0:2.14-1.el7
   python-requests.noarch 0:2.10.0-1.el7                            python-simplejson.x86_64 0:3.5.3-5.el7
   python-six.noarch 0:1.9.0-2.el7                                  python-stevedore.noarch 0:1.8.0-1.el7
   python-tablib.noarch 0:0.10.0-1.el7                              python-testtools.noarch 0:1.8.0-2.el7
   python-traceback2.noarch 0:1.4.0-2.el7                           python-unicodecsv.noarch 0:0.14.1-1.el7
   python-unittest2.noarch 0:1.0.1-1.el7                            python-urllib3.noarch 0:1.15.1-2.el7
   python-warlock.noarch 0:1.0.1-1.el7                              python-webob.noarch 0:1.4.1-2.el7
   python-wrapt.x86_64 0:1.10.5-3.el7                               python2-appdirs.noarch 0:1.4.0-4.el7
   python2-babel.noarch 0:2.3.4-1.el7                               python2-cffi.x86_64 0:1.5.2-1.el7
   python2-cryptography.x86_64 0:1.2.1-3.el7                        python2-debtcollector.noarch 0:0.8.0-1.el7
   python2-iso8601.noarch 0:0.1.11-1.el7                            python2-os-client-config.noarch 0:1.7.4-1.el7
   python2-oslo-config.noarch 2:2.4.0-1.el7                         python2-oslo-i18n.noarch 0:2.6.0-1.el7
   python2-oslo-serialization.noarch 0:1.9.0-1.el7                  python2-oslo-utils.noarch 0:2.5.0-1.el7
   python2-pyasn1.noarch 0:0.1.9-6.el7.1                            python2-pysocks.noarch 0:1.5.6-3.el7
   python2-setuptools.noarch 0:22.0.5-1.el7                         pytz.noarch 0:2012d-5.el7

 Complete!
 ========<

```

## SELinux の設定

OpenStackは、openstack-selinux パッケージにより、OpenStack サービスのセキュリティーポリシーを自動的に管理する


- openstack-selinux パッケージ インストール  [対象: controller01, compute01]

```
# yum install -y openstack-selinux
========>
Installed:
  openstack-selinux.noarch 0:0.6.57-1.el7

Dependency Installed:
  audit-libs-python.x86_64 0:2.4.1-5.el7      checkpolicy.x86_64 0:2.1.12-6.el7              libcgroup.x86_64 0:0.41-8.el7
  libselinux-python.x86_64 0:2.2.2-6.el7      libsemanage-python.x86_64 0:2.1.10-18.el7      policycoreutils-python.x86_64 0:2.2.5-20.el7
  python-IPy.noarch 0:0.75-6.el7              setools-libs.x86_64 0:3.3.7-46.el7

Complete!
========<
```
