# 03_OS事前準備

**自分のリポジトリOpenStack on Ubuntu on オンプレ のコピッたまま。OpenStack on CentOS on ESXi 用に書き換え中。**

ネットワーク設定、DNS(今回はhosts)、NTP 等のOS周りの設定をする


## Network周りの設定

### ssh ログイン

- ログイン
  - 操作PC のTera term から controller01(192.168.101.11) と compute01(192.168.101.21) に SSHログイン</br>
  ユーザー名/パスワード: root/Password123$



### NetworkManagerサービス の無効化 と Networkサービス の有効化

  - NetworkManagerサービスの無効化

  ```
  # systemctl disable NetworkManager
  # systemctl stop NetworkManager
  ```

  - Networkサービス の有効化
  ```
  # systemctl enable network
  # chkconfig --list
  # systemctl start network
  ```


### DEVICE名の確認

```
# <以降の手順にて、ここで確認したデバイス名(eno16780032 と eno33559296)で読み替えて実施すること>
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
NAME=eth0
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
NAME=eth1
UUID=ab9e9c54-a4df-4e00-b2ea-c3e1ca969c77
DEVICE=eno33559296
ONBOOT=yes
PEERDNS=yes
PEERROUTES=yes
========<
```

### パブリックインターフェース の設定

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
NAME=eth1
UUID=ab9e9c54-a4df-4e00-b2ea-c3e1ca969c77
DEVICE=eno33559296
ONBOOT=yes
PEERDNS=yes
PEERROUTES=yes
========<
```


### 設定の反映

```
# reboot
```


## 名前解決

### hostsの設定 (DNSの変わり)

- hostsの設定
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
[対象: controller01, compute01, cli01]

  - hosts動作確認

```
# ping -c 4 controller01
# ping -c 4 compute01
```

- インターネット接続確認

```
# ping -c 4 openstack.org
```

## ファイアウォールの無効化

```
# systemctl disable firewalld
# systemctl stop firewalld
```

## その他確認

```
# hostname

# localectl status
========>
System Locale: LANG=en_US.UTF-8
    VC Keymap: jp
   X11 Layout: jp
========<

timedatectl status
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

- Chrony インストール 設定
[対象: controller01 のみ]

```
# yum -y install chrony

# rpm -qa | grep chrony
========>
chrony-2.1.1-1.el7.centos.x86_64
========<

# vi /etc/chrony.conf
========> 以下追加しなきゃと思うが、、、、、、、、。後で。
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
210 Number of sources = 4
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^- 45.32.43.46.vultr.com         2   6    17    21    -28ms[  -28ms] +/-  380ms
^- y.ns.gin.ntt.net              2   6    17    22  +4790us[+4790us] +/-  125ms
^+ nipper.paina.jp               2   6    17    22   -629us[ -961us] +/-   15ms
^* extendwings.com               2   6    17    22   -574us[ -898us] +/-   13ms
========<
```


### NTPクライアント設定

- Chrony インストール 設定
[対象: compute01]

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

========<
```

## リポジトリ確認

CentOSの場合は、デフォルトで含まれるExtrasにRDO関連のパッケージが含まれている

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

が、ReadHatの場合は、別途追加が必要
