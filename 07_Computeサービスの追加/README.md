# Computeサービスの追加

http://docs.openstack.org/liberty/ja/install-guide-rdo/nova.html



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


- novaデータベース の作成 [対象: controller01]

```
MariaDB [(none)]> CREATE DATABASE nova;
========>
Query OK, 1 row affected (0.01 sec)
========<
```


- novaデータベース へのアクセス権付与　[対象: controller01]

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'Password123$';
========>
Query OK, 0 rows affected (0.00 sec)
========<


MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'Password123$';
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
source ~/admin-openrc.sh
```


### サービスクレデンシャルの作成

- ユーザー作成 [対象: controller01]

```
# openstack user create --domain default --password-prompt nova
========>
User Password:
Repeat User Password:
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | default                          |
| enabled   | True                             |
| id        | 723aea6bd3cb436b84885fc6dccf399f |
| name      | nova                             |
+-----------+----------------------------------+
========<
```


-  ロールの割り当て [対象: controller01]

adminロールを novaユーザーと serviceプロジェクト に追加する

```
# openstack role add --project service --user nova admin
```


- novaサービスエンティティー の作成 [対象: controller01]

```
# openstack service create --name nova --description "OpenStack Compute" compute
========>
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | bf955d0bdef74100848c95b4daa1e620 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
========<
```


### APIエンドポイント の作成

-  public, internal, admin 用の APIエンドポイント の作成 [対象: controller01]

```
# openstack endpoint create --region RegionOne compute public http://controller01:8774/v2/%\(tenant_id\)s
========>
+--------------+-------------------------------------------+
| Field        | Value                                     |
+--------------+-------------------------------------------+
| enabled      | True                                      |
| id           | cfc702302cc64738b73b8b52824fdb95          |
| interface    | public                                    |
| region       | RegionOne                                 |
| region_id    | RegionOne                                 |
| service_id   | bf955d0bdef74100848c95b4daa1e620          |
| service_name | nova                                      |
| service_type | compute                                   |
| url          | http://controller01:8774/v2/%(tenant_id)s |
+--------------+-------------------------------------------+
========<


# openstack endpoint create --region RegionOne compute internal http://controller01:8774/v2/%\(tenant_id\)s
========>
+--------------+-------------------------------------------+
| Field        | Value                                     |
+--------------+-------------------------------------------+
| enabled      | True                                      |
| id           | aea67c1cd26b48d780c5253fdf7c1bce          |
| interface    | internal                                  |
| region       | RegionOne                                 |
| region_id    | RegionOne                                 |
| service_id   | bf955d0bdef74100848c95b4daa1e620          |
| service_name | nova                                      |
| service_type | compute                                   |
| url          | http://controller01:8774/v2/%(tenant_id)s |
+--------------+-------------------------------------------+
========<


# openstack endpoint create --region RegionOne compute admin http://controller01:8774/v2/%\(tenant_id\)s
========>
+--------------+-------------------------------------------+
| Field        | Value                                     |
+--------------+-------------------------------------------+
| enabled      | True                                      |
| id           | 4ac46d79c8a24b82b4403baf2e5158fe          |
| interface    | admin                                     |
| region       | RegionOne                                 |
| region_id    | RegionOne                                 |
| service_id   | bf955d0bdef74100848c95b4daa1e620          |
| service_name | nova                                      |
| service_type | compute                                   |
| url          | http://controller01:8774/v2/%(tenant_id)s |
+--------------+-------------------------------------------+
========<
```



## コンポーネントのインストール と 設定 (コントローラーノード)

### パッケージのインストール (コントローラーノード)

- インストール [対象: controller01]

```
# yum install -y openstack-nova-api openstack-nova-cert openstack-nova-conductor openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler python-novaclient
========>
(省略)
Installed:
  openstack-nova-api.noarch 1:12.0.4-1.el7     openstack-nova-cert.noarch 1:12.0.4-1.el7       openstack-nova-conductor.noarch 1:12.0.4-1.el7
  openstack-nova-console.noarch 1:12.0.4-1.el7 openstack-nova-novncproxy.noarch 1:12.0.4-1.el7 openstack-nova-scheduler.noarch 1:12.0.4-1.el7

Dependency Installed:
  jbigkit-libs.x86_64 0:2.0-11.el7                                          libjpeg-turbo.x86_64 0:1.2.90-5.el7
  libtiff.x86_64 0:4.0.3-25.el7_2                                           libwebp.x86_64 0:0.3.0-3.el7
  libxslt.x86_64 0:1.1.28-5.el7                                             novnc.noarch 0:0.5.1-2.el7
  openstack-nova-common.noarch 1:12.0.4-1.el7                               python-cheetah.x86_64 0:2.4.4-5.el7.centos
  python-ecdsa.noarch 0:0.11-3.el7.centos                                   python-jinja2.noarch 0:2.7.2-2.el7
  python-lxml.x86_64 0:3.2.1-4.el7                                          python-markdown.noarch 0:2.4.1-1.el7.centos
  python-nova.noarch 1:12.0.4-1.el7                                         python-oslo-rootwrap.noarch 0:2.3.0-1.el7
  python-oslo-versionedobjects.noarch 0:0.10.0-1.el7                        python-paramiko.noarch 0:1.15.1-1.el7
  python-pillow.x86_64 0:2.0.0-19.gitd1c6db8.el7                            python-psutil.x86_64 0:1.2.1-1.el7
  python-pygments.noarch 0:2.0.2-4.el7                                      python-rfc3986.noarch 0:0.2.0-1.el7
  python-websockify.noarch 0:0.8.0-1.el7                                    python2-funcsigs.noarch 0:0.4-2.el7
  python2-mock.noarch 0:1.3.0-2.el7                                         python2-os-brick.noarch 0:0.5.0-1.el7
  python2-oslo-reports.noarch 0:0.5.0-1.el7

Complete!
========<
```

### 設定 (コントローラーノード)

- `nova.conf`の設定 [対象: controller01]

  - 補足：
    - [database] セクションで、データベースのアクセス方法を設定
    - [DEFAULT] と [oslo_messaging_rabbit] セクションに、RabbitMQ メッセージキューのアクセス方法を設定
    - [DEFAULT] セクションと [keystone_authtoken] セクションに、Identity サービスへのアクセス方法を設定
    - [DEFAULT] セクションで、コントローラーノードの管理インターフェースの IP アドレスを使用するように``my_ip`` オプションを設定
    - [DEFAULT] セクションで、Networking サービスのサポートを有効化
      - デフォルトで、Compute は組み込みのファイアウォールサービスを使用する。Networking サービスがファイアウォールサービスを提供するため、nova.virt.firewall.NoopFirewallDriver ファイアウォールドライバーを使用して、Compute のファイアウォールサービスを無効化する必要がある。
    - [vnc] セクションで、コントローラーノードの管理インターフェース IP アドレスを使用するように、VNC プロキシーを設定
    - [glance] セクションに、Image service の場所を設定
    - [oslo_concurrency] セクションにロックパスを設定
    - [DEFAULT] セクションで、EC2 API を無効化
    - (オプション) トラブルシューティングしやすくするために、冗長ロギングを [DEFAULT] セクションで有効化

```
# vi /etc/nova/nova.conf
========> 以下を参考に編集
[database]
connection = mysql://nova:Password123$@controller01/nova

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
username = nova
password = Password123$
# <[keystone_authtoken]セクションにある他のオプションがあれば、コメントアウトまたは削除する>


[DEFAULT]
my_ip = 192.168.101.11


[DEFAULT]
network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.NeutronLinuxBridgeInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver


[vnc]
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip


[glance]
host = controller01


[oslo_concurrency]
lock_path = /var/lib/nova/tmp


[DEFAULT]
enabled_apis=osapi_compute,metadata


# <以下、オプション>
# <本手順では追加しない>
[DEFAULT]
...
verbose = True
========<
```


## データベースへの展開 (コントローラーノード)

- データベースへの展開 [対象: controller01]

```
# su -s /bin/sh -c "nova-manage db sync" nova
```


## データベースへの展開 確認 (コントローラーノード)

- 確認 [対象: controller01]

```
[root@controller01 ~]# mysql -u nova -h controller01 -p
Enter password: Password123$
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 21
Server version: 5.5.50-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.



MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| nova               |
+--------------------+
2 rows in set (0.00 sec)


MariaDB [(none)]> USE nova;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [nova]> SHOW TABLES;
+--------------------------------------------+
| Tables_in_nova                             |
+--------------------------------------------+
| agent_builds                               |
| aggregate_hosts                            |
| aggregate_metadata                         |
| aggregates                                 |
| block_device_mapping                       |
| bw_usage_cache                             |
| cells                                      |
| certificates                               |
| compute_nodes                              |
| console_pools                              |
| consoles                                   |
| dns_domains                                |
| fixed_ips                                  |
| floating_ips                               |
| instance_actions                           |
| instance_actions_events                    |
| instance_extra                             |
| instance_faults                            |
| instance_group_member                      |
| instance_group_policy                      |
| instance_groups                            |
| instance_id_mappings                       |
| instance_info_caches                       |
| instance_metadata                          |
| instance_system_metadata                   |
| instance_type_extra_specs                  |
| instance_type_projects                     |
| instance_types                             |
| instances                                  |
| key_pairs                                  |
| migrate_version                            |
| migrations                                 |
| networks                                   |
| pci_devices                                |
| project_user_quotas                        |
| provider_fw_rules                          |
| quota_classes                              |
| quota_usages                               |
| quotas                                     |
| reservations                               |
| s3_images                                  |
| security_group_default_rules               |
| security_group_instance_association        |
| security_group_rules                       |
| security_groups                            |
| services                                   |
| shadow_agent_builds                        |
| shadow_aggregate_hosts                     |
| shadow_aggregate_metadata                  |
| shadow_aggregates                          |
| shadow_block_device_mapping                |
| shadow_bw_usage_cache                      |
| shadow_cells                               |
| shadow_certificates                        |
| shadow_compute_nodes                       |
| shadow_console_pools                       |
| shadow_consoles                            |
| shadow_dns_domains                         |
| shadow_fixed_ips                           |
| shadow_floating_ips                        |
| shadow_instance_actions                    |
| shadow_instance_actions_events             |
| shadow_instance_extra                      |
| shadow_instance_faults                     |
| shadow_instance_group_member               |
| shadow_instance_group_policy               |
| shadow_instance_groups                     |
| shadow_instance_id_mappings                |
| shadow_instance_info_caches                |
| shadow_instance_metadata                   |
| shadow_instance_system_metadata            |
| shadow_instance_type_extra_specs           |
| shadow_instance_type_projects              |
| shadow_instance_types                      |
| shadow_instances                           |
| shadow_key_pairs                           |
| shadow_migrate_version                     |
| shadow_migrations                          |
| shadow_networks                            |
| shadow_pci_devices                         |
| shadow_project_user_quotas                 |
| shadow_provider_fw_rules                   |
| shadow_quota_classes                       |
| shadow_quota_usages                        |
| shadow_quotas                              |
| shadow_reservations                        |
| shadow_s3_images                           |
| shadow_security_group_default_rules        |
| shadow_security_group_instance_association |
| shadow_security_group_rules                |
| shadow_security_groups                     |
| shadow_services                            |
| shadow_snapshot_id_mappings                |
| shadow_snapshots                           |
| shadow_task_log                            |
| shadow_virtual_interfaces                  |
| shadow_volume_id_mappings                  |
| shadow_volume_usage_cache                  |
| snapshot_id_mappings                       |
| snapshots                                  |
| tags                                       |
| task_log                                   |
| virtual_interfaces                         |
| volume_id_mappings                         |
| volume_usage_cache                         |
+--------------------------------------------+
105 rows in set (0.00 sec)

MariaDB [nova]> exit
Bye
[root@controller01 ~]#
```



## Computeサービスの自動起動設定 と 起動 (コントローラーノード)

- 自動起動設定 と 起動 [対象: controller01]

```
# systemctl enable openstack-nova-api.service openstack-nova-cert.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
========>
openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-nova-api.service to /usr/lib/systemd/system/openstack-nova-api.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-nova-cert.service to /usr/lib/systemd/system/openstack-nova-cert.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-nova-consoleauth.service to /usr/lib/systemd/system/openstack-nova-consoleauth.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-nova-scheduler.service to /usr/lib/systemd/system/openstack-nova-scheduler.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-nova-conductor.service to /usr/lib/systemd/system/openstack-nova-conductor.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-nova-novncproxy.service to /usr/lib/systemd/system/openstack-nova-novncproxy.service.
========<

# systemctl start openstack-nova-api.service openstack-nova-cert.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
```



## コンポーネントのインストール と 設定 (コンピュートノード)

上記、コントローラーノードでの作業

以下、コンピュートノードでの作業

KVM 拡張を持つ QEMU ハイパーバイザーを使用

### パッケージのインストール (コンピュートノード)

- インストール [対象: compute01]

```
# yum install -y openstack-nova-compute sysfsutils
========>
Installed:
  openstack-nova-compute.noarch 1:12.0.4-1.el7                          sysfsutils.x86_64 0:2.1.0-16.el7

Dependency Installed:
  MySQL-python.x86_64 0:1.2.3-11.el7                               OpenIPMI-modalias.x86_64 0:2.0.19-11.el7
  attr.x86_64 0:2.4.46-12.el7                                      augeas-libs.x86_64 0:1.4.0-2.el7
  (省略)
  syslinux.x86_64 0:4.05-12.el7                                    syslinux-extlinux.x86_64 0:4.05-12.el7
  tcp_wrappers.x86_64 0:7.6-77.el7                                 unbound-libs.x86_64 0:1.4.20-26.el7
  urw-fonts.noarch 0:2.4-16.el7                                    usbredir.x86_64 0:0.6-7.el7
  xorg-x11-font-utils.x86_64 1:7.5-20.el7                          yajl.x86_64 0:2.0.4-4.el7
  yum-utils.noarch 0:1.1.31-34.el7

Complete!
========<
```

### 設定

- `nova.conf`の設定 [対象: compute01]
  - 補足：
    - [DEFAULT] と [oslo_messaging_rabbit] セクションに、RabbitMQ メッセージキューのアクセス方法を設定
    - [DEFAULT] セクションと [keystone_authtoken] セクションに、Identity サービスへのアクセス方法を設定
    - [DEFAULT] セクションに my_ip オプションを設定
    - [DEFAULT] セクションで、Networking サービスのサポートを有効
      - デフォルトで、Compute は組み込みのファイアウォールサービスを使用する。Networking サービスがファイアウォールサービスを提供するため、nova.virt.firewall.NoopFirewallDriver ファイアウォールドライバーを使用して、Compute のファイアウォールサービスを無効化する必要がある。
    - [vnc] セクションで、リモートコンソールアクセスを有効化
      - サーバーコンポーネントは、すべての IP アドレスをリッスンする。プロキシーコンポーネントはコンピュートノードの管理インターフェースの IP アドレスのみをリッスンする。ベース URL は、Web ブラウザーがこのコンピュートノードにあるインスタンスのリモートコンソールにアクセスするための場所を示す。
    - [glance] セクションに、Image service の場所を設定
    - [oslo_concurrency] セクションにロックパスを設定
    - (オプション) トラブルシューティングしやすくするために、冗長ロギングを [DEFAULT] セクションで有効化

```
# vi /etc/nova/nova.conf
========>
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
username = nova
password = Password123$

[DEFAULT]
my_ip = 192.168.101.21

[DEFAULT]
network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.NeutronLinuxBridgeInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[vnc]
enabled = true
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://controller01:6080/vnc_auto.html

[glance]
host = controller01

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

# <以下オプション>
# <本手順では追加しない>
[DEFAULT]
verbose = True
========<
```

### 仮想マシンハードウェア支援機能サポートの確認

- コンピュートノードの仮想マシンハードウェア支援機能サポートの確認 [対象: compute01]

```
# <1以上の数を返すこと>
# egrep -c '(vmx|svm)' /proc/cpuinfo
========>
4
========<
```

<!--
[root@compute01 ~]# cat /proc/cpuinfo | grep vmx
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts mmx fxsr sse sse2 ss ht syscall nx rdtscp lm constant_tsc arch_perfmon pebs bts nopl xtopology tsc_reliable nonstop_tsc aperfmperf pni vmx ssse3 cx16 sse4_1 sse4_2 x2apic popcnt tsc_deadline_timer hypervisor lahf_lm ida dtherm tpr_shadow vnmi ept vpid tsc_adjust
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts mmx fxsr sse sse2 ss ht syscall nx rdtscp lm constant_tsc arch_perfmon pebs bts nopl xtopology tsc_reliable nonstop_tsc aperfmperf pni vmx ssse3 cx16 sse4_1 sse4_2 x2apic popcnt tsc_deadline_timer hypervisor lahf_lm ida dtherm tpr_shadow vnmi ept vpid tsc_adjust
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts mmx fxsr sse sse2 ss ht syscall nx rdtscp lm constant_tsc arch_perfmon pebs bts nopl xtopology tsc_reliable nonstop_tsc aperfmperf pni vmx ssse3 cx16 sse4_1 sse4_2 x2apic popcnt tsc_deadline_timer hypervisor lahf_lm ida dtherm tpr_shadow vnmi ept vpid tsc_adjust
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts mmx fxsr sse sse2 ss ht syscall nx rdtscp lm constant_tsc arch_perfmon pebs bts nopl xtopology tsc_reliable nonstop_tsc aperfmperf pni vmx ssse3 cx16 sse4_1 sse4_2 x2apic popcnt tsc_deadline_timer hypervisor lahf_lm ida dtherm tpr_shadow vnmi ept vpid tsc_adjust
-->



#### 上記コマンドで0を返した場合のみ以下を実施、*1以上の場合は実施しない*

- [01_ESXi事前準備](https://github.com/Soichiro75/openstack-liberty-on-centos-japanese/tree/master/01_ESXi事前準備) と、[02_OSインストール](https://github.com/Soichiro75/openstack-liberty-on-centos-japanese/tree/master/02_OSインストール) の以下を実施

  - ESXiホストのBIOSにて`Virtualization Technology(VT-x or AMD-V) を有効化`
  - VM Network 101とVM Network 102両ポートグループに、`プロミスキャス(無差別)モード 承諾`(OpenStack上のVMと通信をとれるようにするため)を設定
  - ESXiホストの `/etc/vmware/config` に `vhv.enable = "TRUE"` を記載
  - compute01の`Intel VT-x/AMD-Vを命令セット仮想化に使用し、ソフトウェアをMMU仮想化に使用の有効`
  - ESXiホスト、compute01 の`再起動` (勿論、controller01も同じく再起動となる)
  - compute01 にて `egrep -c '(vmx|svm)' /proc/cpuinfo` が`1以上`を返すかリトライ
  - それでも0だった場合は、以下を実施

```
上記のコマンドが 0 を返す場合、コンピュートノードはハードウェア支援機能をサポートしていない。libvirt が KVM の代わりに QEMU を使用するように設定する必要がある。

    (補足)：
    libvirt とは、仮想化管理用の共通APIを提供する、レッドハットを中心としたオープンソースプロジェクト。仮想機械の制御を抽象化したライブラリ。 本ライブラリの特徴は、サポート範囲が広いことである。 サポートしている仮想化は、現在Xen、KVM、QEMU、LXC、OpenVZ、UML、VirtualBox、VMware ESX・GSX・Workstation・Player、Hyper-V、そしてクラスタ管理ソフトOpenNebulaである。

/etc/nova/nova.conf ファイルの [libvirt] セクションを以下のように編集します。

[libvirt]
...
virt_type = qemu

```

### Computeサービスの自動起動設定 と 起動

- 自動起動設定 と 起動 [対象: compute01]

```
# systemctl enable libvirtd.service openstack-nova-compute.service
========>
========<

# systemctl start libvirtd.service openstack-nova-compute.service
```
