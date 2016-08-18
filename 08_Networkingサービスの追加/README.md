# 08_Networkingサービスの追加

http://docs.openstack.org/liberty/ja/install-guide-rdo/neutron.html

OpenStack Networking (neutron) は、 OpenStack 環境で、仮想ネットワークインフラストラクチャ (VNI) のすべての面と、物理ネットワークインフラストラクチャ (PNI) のアクセス層の側面を管理する。 OpenStack Networking を使うと、プロジェクトが ファイアウォール 、負荷分散装置 、VPN などの高度な仮想ネットワークトポロジーを作成できるようになる。(以下のオプション2 セルフサービスネットワーク で構築した場合。)


Networking サービスは、オプション 1 と 2 の 2 つのアーキテクチャーのいずれかを使ってデプロイできる。

オプション 1: 可能な限り最も単純なアーキテクチャーをデプロイします。パブリック (プロバイダー) ネットワークへのインスタンスの接続のみに対応します。セルフサービスのネットワークやルーター、Floating IP アドレスはサポートされません。 admin や他の特権ユーザーだけがプロバイダーネットワークを管理できます。

オプション 2: オプション 1 にレイヤー 3 サービスを組み合わせたもので、セルフサービス (プライベート) ネットワークへのインスタンスの接続をサポートします。 demo や他の非特権ユーザーがセルフサービスネットワークを管理し、それにはセルフサービスネットワークやプロバイダーネットワークの間の接続性を提供するルーターも含まれます。また、 Floating IP アドレスにより、セルフサービスネットワークに接続されたインスタンスへの、インターネットなどの外部ネットワークからの接続性が提供されます。

この手順書では、「オプション2 セルフサービスネットワーク」で構築する。


## 事前準備 データベース、サービスクレデンシャル、API エンドポイント の作成

### データベースの作成

- rootでデータベースに接続 [対象: controller01]

```
# mysql -u root -p
========>
Enter password: Password123$
MariaDB [(none)]>
========<
```


- neutronデータベース の作成 [対象: controller01]

```
MariaDB [(none)]> CREATE DATABASE neutron;
========>
Query OK, 1 row affected (0.00 sec)
========<
```


- neutronデータベース へのアクセス権付与　[対象: controller01]

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'Password123$';
========>
Query OK, 0 rows affected (0.00 sec)
========<


MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'Password123$';
========>
Query OK, 0 rows affected (0.00 sec)
========<
```


- データベースを抜ける[対象: controller01]

```
MariaDB [(none)]> exit
========>
Bye
[root@controller01 ~]#
========<
```


### アクセス権の取得

- adminクレデンシャルの読み込み [対象: controller01]

管理者専用 CLI コマンドへのアクセス権を読み込む

```
# source ~/admin-openrc.sh
```


### サービスクレデンシャルの作成

- ユーザー作成 [対象: controller01]

```
# openstack user create --domain default --password-prompt neutron
========>
User Password:  Password123$  <== 入力中は表示されない
Repeat User Password:  Password123$  <== 入力中は表示されない
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | default                          |
| enabled   | True                             |
| id        | e7c31b5c2c8049d3adf99c200e726e62 |
| name      | neutron                          |
+-----------+----------------------------------+
========<
```


-  ロールの割り当て [対象: controller01]

adminロールを neutronユーザーと serviceプロジェクト に追加する

```
# openstack role add --project service --user neutron admin
```


- neutronサービスエンティティー の作成 [対象: controller01]

```
# openstack service create --name neutron --description "OpenStack Networking" network
========>
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | 1668d79791bc4badbce0761248df406c |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
========<
```


### APIエンドポイント の作成

-  public, internal, admin 用の APIエンドポイント の作成 [対象: controller01]

```
# openstack endpoint create --region RegionOne network public http://controller01:9696
========>
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 1872aa1a588d466c88a16d64089cf0d0 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 1668d79791bc4badbce0761248df406c |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller01:9696         |
+--------------+----------------------------------+
========<


# openstack endpoint create --region RegionOne network internal http://controller01:9696
========>
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | c8e419f758ab4c8ba6228e9fb763112a |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 1668d79791bc4badbce0761248df406c |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller01:9696         |
+--------------+----------------------------------+
========<


# openstack endpoint create --region RegionOne network admin http://controller01:9696
========>
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | ebc00b8dc0a44a51a8131321d972cf7e |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 1668d79791bc4badbce0761248df406c |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller01:9696         |
+--------------+----------------------------------+
========<
```



## コンポーネントのインストール と 設定 (コントローラーノード)

[「オプション2 セルフサービスネットワーク」](http://docs.openstack.org/liberty/ja/install-guide-rdo/neutron-controller-install-option2.html) にて構築する


### パッケージのインストール (コントローラーノード)

- インストール [対象: controller01]

```
# yum install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge python-neutronclient ebtables ipset
========>
(省略)
Installed:
  ebtables.x86_64 0:2.0.10-13.el7                             ipset.x86_64 0:6.19-4.el7
  openstack-neutron.noarch 1:7.1.1-1.el7                      openstack-neutron-linuxbridge.noarch 1:7.1.1-1.el7
  openstack-neutron-ml2.noarch 1:7.1.1-1.el7

Dependency Installed:
  bridge-utils.x86_64 0:1.5-9.el7                                 conntrack-tools.x86_64 0:1.4.2-9.el7
  dibbler-client.x86_64 0:1.0.1-0.RC1.2.el7                       dnsmasq-utils.x86_64 0:2.66-14.el7_1
  ipset-libs.x86_64 0:6.19-4.el7                                  keepalived.x86_64 0:1.2.13-7.el7
  libnetfilter_cthelper.x86_64 0:1.0.0-8.el7                      libnetfilter_cttimeout.x86_64 0:1.0.0-6.el7
  libnetfilter_queue.x86_64 0:1.0.2-2.el7                         libxml2-python.x86_64 0:2.9.1-6.el7_2.3
  libxslt-python.x86_64 0:1.1.28-5.el7                            lm_sensors-libs.x86_64 0:3.3.4-11.el7
  net-snmp-agent-libs.x86_64 1:5.7.2-24.el7_2.1                   net-snmp-libs.x86_64 1:5.7.2-24.el7_2.1
  openstack-neutron-common.noarch 1:7.1.1-1.el7                   python-logutils.noarch 0:0.3.3-3.el7
  python-ncclient.noarch 0:0.4.2-2.el7                            python-neutron.noarch 1:7.1.1-1.el7
  python-ryu.noarch 0:3.30-1.el7                                  python-webtest.noarch 0:1.3.4-6.el7
  python2-pecan.noarch 0:1.0.2-2.el7                              python2-singledispatch.noarch 0:3.4.0.3-4.el7
  radvd.x86_64 0:1.9.2-9.el7

Complete!
========<
```


### 設定 (コントローラーノード)

- `neutron.conf`の設定 [対象: controller01]
  - 補足:
    - [database] セクションで、データベースのアクセス方法を設定
    - [DEFAULT] セクションで、Modular Layer 2 (ML2) プラグイン、ルーターサービス、IP アドレス重複を有効化
    - [DEFAULT] と [oslo_messaging_rabbit] セクションに、RabbitMQ メッセージキューのアクセス方法を設定
    - [DEFAULT] セクションと [keystone_authtoken] セクションに、Identity サービスへのアクセス方法を設定
    - [DEFAULT] セクションと [nova] セクションで、Networking が Compute にネットワークトポロジーの変更を通知するよう設定
    - [oslo_concurrency] セクションにロックパスを設定
    - (オプション) トラブルシューティングしやすくするために、冗長ロギングを [DEFAULT] セクションで有効
    -

```
# vi /etc/neutron/neutron.conf
========> 以下を参考に編集
[database]
connection = mysql://neutron:Password123$@controller01/neutron


[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True


[DEFAULT]
rpc_backend = rabbit

[oslo_messaging_rabbit]
rabbit_host = controller01
rabbit_userid = openstack
rabbit_password = Password123$


[DEFAULT]
auth_strategy = keystone

[keystone_authtoken]
auth_uri = http://controller01:5000
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = Password123$
# <[keystone_authtoken] セクションにある他のオプションは、コメントアウト>


[DEFAULT]
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
nova_url = http://controller01:8774/v2

[nova]
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
region_name = RegionOne
project_name = service
username = nova
password = Password123$


[oslo_concurrency]
lock_path = /var/lib/neutron/tmp

# <オプション>
# <本手順では実施しない>
[DEFAULT]
verbose = True
========<
```


- ML2 プラグインの設定 [対象: controller01]
  - 補足:
    - [ml2] セクションで、フラット、VLAN、VXLAN ネットワークを有効化
    - [ml2] セクションで、VXLAN プロジェクト (プライベート) ネットワークを有効化
    - [ml2] セクションで、Linux ブリッジ機構および layer-2 population 機構を有効化
    - [ml2] セクションで、ポートセキュリティー拡張ドライバーを有効化
    - [ml2_type_flat] セクションで、パブリックフラットプロバイダーネットワークを設定
    - [ml2_type_vxlan] セクションで、プライベートネットワーク用の VXLAN ネットワーク ID 範囲を設定
    - [securitygroup] セクションで、 ipset を有効にし、セキュリティーグループルールの効率性を向上
     - ipset: 連続する IP アドレスの全体に一致するファイアウォールルールを作成できる、iptables の拡張。これらのセットは、効率化するためにインデックス化されたデータ構造、とくに大量のルールを持つシステムにある。

```
vi /etc/neutron/plugins/ml2/ml2_conf.ini
========>
[ml2]
type_drivers = flat,vlan,vxlan


[ml2]
tenant_network_types = vxlan


[ml2]
mechanism_drivers = linuxbridge,l2population


[ml2]
extension_drivers = port_security


[ml2_type_flat]
flat_networks = public


[ml2_type_vxlan]
vni_ranges = 1:1000


[securitygroup]
enable_ipset = True
========<
```


- Linux ブリッジエージェントの設定 [対象: controller01]

Linux ブリッジエージェントは、プライベートネットワーク向けの VXLAN トンネルなどの、インスタンス用の L2 (ブリッジおよびスイッチ) 仮想ネットワークインフラを構築して、セキュリティーグループを処理する

  - 補足:
    - [linux_bridge] セクションにおいて、仮想パブリックネットワークを物理パブリックネットワークのインターフェースに対応付ける
    - [vxlan] セクションにおいて、VXLAN オーバーレイネットワークを有効にし、オーバーレイネットワークを処理する物理ネットワークインターフェースの IP アドレスを設定し、layer-2 population を有効化
    - [agent] セクションにおいて、ARP スプーフィングの保護を有効化
    - [securitygroup] セクションで、セキュリティグループを有効にし、 Linux ブリッジ iptables ファイアウォールドライバーを設定
    - PUBLIC_INTERFACE(老番のIPを振っていないインターフェース eno33559296)</br>
    OVERLAY_INTERFACE(若番のIPを振っているインターフェース eno16780032)(本手順では管理インターフェースと兼用している)

```
# vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini
========>
[linux_bridge]
physical_interface_mappings = public:eno33559296


[vxlan]
enable_vxlan = True
local_ip = 192.168.101.11
l2_population = True


[agent]
prevent_arp_spoofing = True


[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
========<
```



- L3 エージェントの設定 [対象: controller01]

L3 エージェント は、仮想ネットワーク用のルーティングおよび NAT サービスを提供する

  - 補足:
    - [DEFAULT] セクションで、Linux ブリッジインターフェースドライバー、外部ネットワークブリッジを設定
      - 以下の設定では、`external_network_bridge=オプションに値を入れていない`。これは、1 つのエージェントで、複数の外部ネットワークを有効にするためであり、意図的に値をいれていない(間違いではない)。
    - (オプション) トラブルシューティングしやすくするために、冗長ロギングを [DEFAULT] セクションで有効化

```
# vi /etc/neutron/l3_agent.ini
========>
[DEFAULT]
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
external_network_bridge =


# <オプション>
# <本手順では追加しない>
[DEFAULT]
verbose = True
========<
```


- DHCP エージェントの設定 [対象: controller01]

DHCP エージェント は、仮想ネットワーク向けに DHCP サービスを提供する

  - 補足:
    - [DEFAULT] セクションにおいて、Linux ブリッジインターフェースドライバー、Dnsmasq DHCP ドライバーを設定して、 isolated metadata を有効にする。これにより、パブリックネットワークにあるインスタンスがネットワーク経由でメタデータにアクセスできる。
    - (オプション) トラブルシューティングしやすくするために、冗長ロギングを [DEFAULT] セクションで有効化。
    - [DEFAULT] セクションで、dnsmasq 設定ファイルを有効化
    -

```
# vi /etc/neutron/dhcp_agent.ini
========>
[DEFAULT]
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True


# <オプション>
# <本手順では追加しない>
[DEFAULT]
verbose = True


[DEFAULT]
dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
========<
```

```
# vi /etc/neutron/dnsmasq-neutron.conf
========> 新規作成
# <DHCP MTU オプション (26) を有効にし、値を 1450 バイトに設定>
dhcp-option-force=26,1450
========<
```



## メタデータエージェントの設定

http://docs.openstack.org/liberty/ja/install-guide-rdo/neutron-controller-install.html#neutron-controller-metadata-agent

メタデータエージェント は、クレデンシャルなどの設定情報をインスタンスに提供する

- メタデータエージェントの設定 [対象: controller01]
  - 補足:
    - [DEFAULT] セクションに、アクセスパラメーターを設定
    - [DEFAULT] セクションに、メタデータホストを設定
    - [DEFAULT] セクションに、メタデータプロキシーの共有シークレットを設定します。
    - (オプション) トラブルシューティングしやすくするために、冗長ロギングを [DEFAULT] セクションで有効化

```
# vi /etc/neutron/metadata_agent.ini
========>
[DEFAULT]
auth_uri = http://controller01:5000
auth_url = http://controller01:35357
auth_region = RegionOne
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = Password123$


[DEFAULT]
nova_metadata_ip = controller01


[DEFAULT]
metadata_proxy_shared_secret = Password123$


# <オプション>
# <本手順では追加しない>
[DEFAULT]
verbose = True
========<
```


## Networking を使用するための Compute の設定

-  Networking を使用するための Compute の設定 [対象: controller01]

  - 補足:
    - [neutron] セクションに、アクセス用のパラメーターを設定し、メタデータプロキシーを有効にし、シークレットを設定

```
# vi /etc/nova/nova.conf
========>
[neutron]
...
url = http://controller01:9696
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
region_name = RegionOne
project_name = service
username = neutron
password = Password123$

service_metadata_proxy = True
metadata_proxy_shared_secret = Password123$
========<
```


## シンボリックリンク作成

Networking のサービス初期化スクリプトは、シンボリックリンク /etc/neutron/plugin.ini が ML2 プラグイン設定ファイル /etc/neutron/plugins/ml2/ml2_conf.ini を指している必要がある

- 初期化スクリプトが使用する シンボリックリンクの作成 [対象: controller01]

```
# <事前確認>
# ls -l /etc/neutron/plugin.ini
========>
ls: cannot access /etc/neutron/plugin.ini: No such file or directory
========<


# <事前確認>
# ls -l /etc/neutron/plugins/ml2/ml2_conf.ini
========>
-rw-r-----. 1 root neutron 5027 Aug 18 14:12 /etc/neutron/plugins/ml2/ml2_conf.ini
========<


# <設定>
# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini


# <確認>
# ls -l /etc/neutron/plugin.ini
========>
lrwxrwxrwx. 1 root root 37 Aug 18 16:20 /etc/neutron/plugin.ini -> /etc/neutron/plugins/ml2/ml2_conf.ini
========<
```


## データベースの展開

- データベースの展開 [対象: controller01]

```
# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
========>
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
  Running upgrade for neutron ...
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> juno, juno_initial
INFO  [alembic.runtime.migration] Running upgrade juno -> 44621190bc02, add_uniqueconstraint_ipavailability_ranges
INFO  [alembic.runtime.migration] Running upgrade 44621190bc02 -> 1f71e54a85e7, ml2_network_segments models change for multi-segment network.
INFO  [alembic.runtime.migration] Running upgrade 1f71e54a85e7 -> 408cfbf6923c, remove ryu plugin
INFO  [alembic.runtime.migration] Running upgrade 408cfbf6923c -> 28c0ffb8ebbd, remove mlnx plugin
INFO  [alembic.runtime.migration] Running upgrade 28c0ffb8ebbd -> 57086602ca0a, scrap_nsx_adv_svcs_models
INFO  [alembic.runtime.migration] Running upgrade 57086602ca0a -> 38495dc99731, ml2_tunnel_endpoints_table
INFO  [alembic.runtime.migration] Running upgrade 38495dc99731 -> 4dbe243cd84d, nsxv
INFO  [alembic.runtime.migration] Running upgrade 4dbe243cd84d -> 41662e32bce2, L3 DVR SNAT mapping
INFO  [alembic.runtime.migration] Running upgrade 41662e32bce2 -> 2a1ee2fb59e0, Add mac_address unique constraint
INFO  [alembic.runtime.migration] Running upgrade 2a1ee2fb59e0 -> 26b54cf9024d, Add index on allocated
INFO  [alembic.runtime.migration] Running upgrade 26b54cf9024d -> 14be42f3d0a5, Add default security group table
INFO  [alembic.runtime.migration] Running upgrade 14be42f3d0a5 -> 16cdf118d31d, extra_dhcp_options IPv6 support
INFO  [alembic.runtime.migration] Running upgrade 16cdf118d31d -> 43763a9618fd, add mtu attributes to network
INFO  [alembic.runtime.migration] Running upgrade 43763a9618fd -> bebba223288, Add vlan transparent property to network
INFO  [alembic.runtime.migration] Running upgrade bebba223288 -> 4119216b7365, Add index on tenant_id column
INFO  [alembic.runtime.migration] Running upgrade 4119216b7365 -> 2d2a8a565438, ML2 hierarchical binding
INFO  [alembic.runtime.migration] Running upgrade 2d2a8a565438 -> 2b801560a332, Remove Hyper-V Neutron Plugin
INFO  [alembic.runtime.migration] Running upgrade 2b801560a332 -> 57dd745253a6, nuage_kilo_migrate
INFO  [alembic.runtime.migration] Running upgrade 57dd745253a6 -> f15b1fb526dd, Cascade Floating IP Floating Port deletion
INFO  [alembic.runtime.migration] Running upgrade f15b1fb526dd -> 341ee8a4ccb5, sync with cisco repo
INFO  [alembic.runtime.migration] Running upgrade 341ee8a4ccb5 -> 35a0f3365720, add port-security in ml2
INFO  [alembic.runtime.migration] Running upgrade 35a0f3365720 -> 1955efc66455, weight_scheduler
INFO  [alembic.runtime.migration] Running upgrade 1955efc66455 -> 51c54792158e, Initial operations for subnetpools
INFO  [alembic.runtime.migration] Running upgrade 51c54792158e -> 589f9237ca0e, Cisco N1kv ML2 driver tables
INFO  [alembic.runtime.migration] Running upgrade 589f9237ca0e -> 20b99fd19d4f, Cisco UCS Manager Mechanism Driver
INFO  [alembic.runtime.migration] Running upgrade 20b99fd19d4f -> 034883111f, Remove allow_overlap from subnetpools
INFO  [alembic.runtime.migration] Running upgrade 034883111f -> 268fb5e99aa2, Initial operations in support of subnet allocation from a pool
INFO  [alembic.runtime.migration] Running upgrade 268fb5e99aa2 -> 28a09af858a8, Initial operations to support basic quotas on prefix space in a subnet pool
INFO  [alembic.runtime.migration] Running upgrade 28a09af858a8 -> 20c469a5f920, add index for port
INFO  [alembic.runtime.migration] Running upgrade 20c469a5f920 -> kilo, kilo
INFO  [alembic.runtime.migration] Running upgrade kilo -> 354db87e3225, nsxv_vdr_metadata.py
INFO  [alembic.runtime.migration] Running upgrade 354db87e3225 -> 599c6a226151, neutrodb_ipam
INFO  [alembic.runtime.migration] Running upgrade 599c6a226151 -> 52c5312f6baf, Initial operations in support of address scopes
INFO  [alembic.runtime.migration] Running upgrade 52c5312f6baf -> 313373c0ffee, Flavor framework
INFO  [alembic.runtime.migration] Running upgrade 313373c0ffee -> 8675309a5c4f, network_rbac
INFO  [alembic.runtime.migration] Running upgrade kilo -> 30018084ec99, Initial no-op Liberty contract rule.
INFO  [alembic.runtime.migration] Running upgrade 30018084ec99, 8675309a5c4f -> 4ffceebfada, network_rbac
INFO  [alembic.runtime.migration] Running upgrade 4ffceebfada -> 5498d17be016, Drop legacy OVS and LB plugin tables
INFO  [alembic.runtime.migration] Running upgrade 5498d17be016 -> 2a16083502f3, Metaplugin removal
INFO  [alembic.runtime.migration] Running upgrade 2a16083502f3 -> 2e5352a0ad4d, Add missing foreign keys
INFO  [alembic.runtime.migration] Running upgrade 2e5352a0ad4d -> 11926bcfe72d, add geneve ml2 type driver
INFO  [alembic.runtime.migration] Running upgrade 11926bcfe72d -> 4af11ca47297, Drop cisco monolithic tables
INFO  [alembic.runtime.migration] Running upgrade 8675309a5c4f -> 45f955889773, quota_usage
INFO  [alembic.runtime.migration] Running upgrade 45f955889773 -> 26c371498592, subnetpool hash
INFO  [alembic.runtime.migration] Running upgrade 26c371498592 -> 1c844d1677f7, add order to dnsnameservers
INFO  [alembic.runtime.migration] Running upgrade 1c844d1677f7 -> 1b4c6e320f79, address scope support in subnetpool
INFO  [alembic.runtime.migration] Running upgrade 1b4c6e320f79 -> 48153cb5f051, qos db changes
INFO  [alembic.runtime.migration] Running upgrade 48153cb5f051 -> 9859ac9c136, quota_reservations
INFO  [alembic.runtime.migration] Running upgrade 9859ac9c136 -> 34af2b5c5a59, Add dns_name to Port
  OK
========<
```

## Compute API サービスを再起動

- Compute API サービスを再起動 [対象: controller01]

```
# systemctl restart openstack-nova-api.service
```


## サービス の自動起動設定 と 起動

- Networkingサービス の自動起動設定 と 起動 [対象: controller01]

```
# systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
========>
[root@controller01 ~]# systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-server.service to /usr/lib/systemd/system/neutron-server.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-linuxbridge-agent.service to /usr/lib/systemd/system/neutron-linuxbridge-agent.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-dhcp-agent.service to /usr/lib/systemd/system/neutron-dhcp-agent.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-metadata-agent.service to /usr/lib/systemd/system/neutron-metadata-agent.service.
========<


# systemctl start neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service


# <本手順では、セルフサービスネットワーク のため、以下も実施>
# systemctl enable neutron-l3-agent.service
========>
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-l3-agent.service to /usr/lib/systemd/system/neutron-l3-agent.service.
========<

# <本手順では、セルフサービスネットワーク のため、以下も実施>
# systemctl start neutron-l3-agent.service
```

</br>
</br>
</br>
</br>
</br>

***

## コンポーネントのインストール

以下、コンピュートノードで実施

- コンポーネントのインストール [対象: compute01]

```
# yum install -y openstack-neutron openstack-neutron-linuxbridge ebtables ipset
========>
Installed:
  ipset.x86_64 0:6.19-4.el7     openstack-neutron.noarch 1:7.1.1-1.el7     openstack-neutron-linuxbridge.noarch 1:7.1.1-1.el7

Dependency Installed:
  conntrack-tools.x86_64 0:1.4.2-9.el7                           dibbler-client.x86_64 0:1.0.1-0.RC1.2.el7
  dnsmasq-utils.x86_64 0:2.66-14.el7_1                           ipset-libs.x86_64 0:6.19-4.el7
  keepalived.x86_64 0:1.2.13-7.el7                               libnetfilter_cthelper.x86_64 0:1.0.0-8.el7
  libnetfilter_cttimeout.x86_64 0:1.0.0-6.el7                    libnetfilter_queue.x86_64 0:1.0.2-2.el7
  lm_sensors-libs.x86_64 0:3.3.4-11.el7                          net-snmp-agent-libs.x86_64 1:5.7.2-24.el7_2.1
  net-snmp-libs.x86_64 1:5.7.2-24.el7_2.1                        openstack-neutron-common.noarch 1:7.1.1-1.el7
  python-logutils.noarch 0:0.3.3-3.el7                           python-neutron.noarch 1:7.1.1-1.el7
  python-oslo-policy.noarch 0:0.11.0-1.el7                       python-ryu.noarch 0:3.30-1.el7
  python-simplegeneric.noarch 0:0.8-7.el7                        python-webtest.noarch 0:1.3.4-6.el7
  python2-pecan.noarch 0:1.0.2-2.el7                             python2-singledispatch.noarch 0:3.4.0.3-4.el7

Complete!
========<
```


## 共通コンポーネントの設定

Networking の共通コンポーネントの設定は、認証メカニズム、メッセージキュー、プラグインがある

- 共通コンポーネントの設定 [対象: compute01]
  - 補足:
    - コンピュートノードはデータベースに直接アクセスしないため、[database] セクションにおいて、すべての connection オプションをコメントアウト
    - [DEFAULT] と [oslo_messaging_rabbit] セクションに、RabbitMQ メッセージキューのアクセス方法を設定
    - [DEFAULT] セクションと [keystone_authtoken] セクションに、Identity サービスへのアクセス方法を設定
    - [oslo_concurrency] セクションにロックパスを設定
    - (オプション) トラブルシューティングしやすくするために、冗長ロギングを [DEFAULT] セクションで有効化

```
# vi /etc/neutron/neutron.conf
========>
# <[database] セクションにおいて、すべての connection オプションをコメントアウト>
# <ただ、デフォルトでコメントアウト済み のため、特に作業は発生しない想定>


[DEFAULT]
rpc_backend = rabbit


[oslo_messaging_rabbit]
rabbit_host = controller01
rabbit_userid = openstack
rabbit_password = Password123$


[DEFAULT]
auth_strategy = keystone


[keystone_authtoken]
auth_uri = http://controller01:5000
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = Passwordd123$


[oslo_concurrency]
lock_path = /var/lib/neutron/tmp

# <オプション>
# <本手順では追加しない>
[DEFAULT]
verbose = True
========<
```


## Linux ブリッジエージェントの設定

Linux ブリッジエージェントは、プライベートネットワーク向けの VXLAN トンネルなどの、インスタンス用の L2 (ブリッジおよびスイッチ) 仮想ネットワークインフラを構築して、セキュリティーグループを処理

- Linux ブリッジエージェントの設定 [対象: compute01]
  - 補足:
    - [linux_bridge] セクションにおいて、仮想パブリックネットワークを物理パブリックネットワークのインターフェースに対応付け
    - [vxlan] セクションにおいて、VXLAN オーバーレイネットワークを有効にし、オーバーレイネットワークを処理する物理ネットワークインターフェースの IP アドレスを設定し、layer-2 population を有効化
    - [agent] セクションにおいて、ARP スプーフィングの保護を有効化
    - [securitygroup] セクションで、セキュリティグループを有効にし、 Linux ブリッジ iptables ファイアウォールドライバーを設定
    - PUBLIC_INTERFACE(老番のIPを振っていないインターフェース eno33559296)</br>
    OVERLAY_INTERFACE(若番のIPを振っているインターフェース eno16780032)(本手順では管理インターフェースと兼用している)


```
# vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini
========>
[linux_bridge]
physical_interface_mappings = public:eno33559296


[vxlan]
enable_vxlan = True
local_ip = 192.168.101.21
l2_population = True


[agent]
prevent_arp_spoofing = True


[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
========<
```


## Networking を使用するための Compute の設定

- Networking を使用するための Compute の設定 [対象: compute01]
  - 補足:
    - [neutron] セクションに、アクセスパラメーターを設定

```
# vi /etc/nova/nova.conf
========>
[neutron]
url = http://controller01:9696
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
region_name = RegionOne
project_name = service
username = neutron
password = Password123$
========<
```

## サービス の再起動 と起動、自動起動

- Compute Service の再起動 [対象: compute01]

```
# systemctl restart openstack-nova-compute.service
```

- Linux ブリッジエージェントを起動し、システム起動時に自動的に起動するよう設定 [対象: compute01]

```
# systemctl enable neutron-linuxbridge-agent.service
========>
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-linuxbridge-agent.service to /usr/lib/systemd/system/neutron-linuxbridge-agent.service.
========<


# systemctl start neutron-linuxbridge-agent.service
```


</br>
</br>
</br>
</br>
</br>

***

## 動作確認

以下、コントローラーノードで実施

- 管理者専用 CLI コマンドへのアクセス権読み込み [対象: controller01]

admin クレデンシャルを読み込む

```
# source admin-openrc.sh
```


- プロセスの起動確認 [対象: controller01]

ロード済み拡張機能一覧を表示して、neutron-server プロセスが正しく起動していることを確認する

```
# neutron ext-list
========>
+-----------------------+-----------------------------------------------+
| alias                 | name                                          |
+-----------------------+-----------------------------------------------+
| dns-integration       | DNS Integration                               |
| ext-gw-mode           | Neutron L3 Configurable external gateway mode |
| binding               | Port Binding                                  |
| agent                 | agent                                         |
| subnet_allocation     | Subnet Allocation                             |
| l3_agent_scheduler    | L3 Agent Scheduler                            |
| external-net          | Neutron external network                      |
| flavors               | Neutron Service Flavors                       |
| net-mtu               | Network MTU                                   |
| quotas                | Quota management support                      |
| l3-ha                 | HA Router extension                           |
| provider              | Provider Network                              |
| multi-provider        | Multi Provider Network                        |
| extraroute            | Neutron Extra Route                           |
| router                | Neutron L3 Router                             |
| extra_dhcp_opt        | Neutron Extra DHCP opts                       |
| security-group        | security-group                                |
| dhcp_agent_scheduler  | DHCP Agent Scheduler                          |
| rbac-policies         | RBAC Policies                                 |
| port-security         | Port Security                                 |
| allowed-address-pairs | Allowed Address Pairs                         |
| dvr                   | Distributed Virtual Router                    |
+-----------------------+-----------------------------------------------+
========<
```



- (セルフサービスネットワーク) neutron エージェントの起動確認 [対象: controller01]

neutron エージェントが正常に起動したことを確認するために、エージェントを一覧表示します。

```
# neutron agent-list
========>
+--------------------------------------+--------------------+--------------+-------+----------------+---------------------------+
| id                                   | agent_type         | host         | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+--------------+-------+----------------+---------------------------+
| 50538e6a-aa7e-4acb-8987-c21e8b2582e9 | Metadata agent     | controller01 | :-)   | True           | neutron-metadata-agent    |
| 66fd76e9-88b1-453f-8f34-481ca2e860ca | L3 agent           | controller01 | :-)   | True           | neutron-l3-agent          |
| ad3e42bc-b27a-4cc8-8fa7-d62874116263 | Linux bridge agent | compute01    | :-)   | True           | neutron-linuxbridge-agent |
| b9e4134c-b7e7-40e4-b683-3964e21c1947 | DHCP agent         | controller01 | :-)   | True           | neutron-dhcp-agent        |
| d751c405-17c0-4048-851b-61bd6dad2181 | Linux bridge agent | controller01 | :-)   | True           | neutron-linuxbridge-agent |
+--------------------------------------+--------------------+--------------+-------+----------------+---------------------------+
========<
