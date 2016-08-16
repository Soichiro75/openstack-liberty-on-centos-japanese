# Imageサービスの追加

OpenStack Image service (glance) により、ユーザーが仮想マシンイメージを検索、登録、取得できるようになる。 REST API が提供され、仮想マシンイメージメタデータを問い合わせり、実際のイメージを取得したり出来る。 Image service 経由で利用可能な仮想マシンイメージは、単なるファイルシステムから OpenStack Object Storage(swift) のようなオブジェクトストレージシステムまで、さまざまな場所に保存出来る。

この手順では Image service が `file` バックエンドを使用するよう設定する。この場合、Image service が動作しているコントローラーノード上のディレクトリー(/var/lib/glance/images/)にイメージがアップロードされ保存される。

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


- glanceデータベース の作成 [対象: controller01]

```
MariaDB [(none)]> CREATE DATABASE glance;
========>
Query OK, 1 row affected (0.01 sec)
========<
```


- glanceデータベース へのアクセス権付与　[対象: controller01]

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'Password123$';
========>
Query OK, 0 rows affected (0.00 sec)
========<


MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'Password123$';
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


- サービスクレデンシャルの作成 [対象: controller01]

```
# openstack user create --domain default --password-prompt glance
========>
User Password: Password123$ <== 入力中は表示されない
Repeat User Password: Password123$  <== 入力中は表示されない
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | default                          |
| enabled   | True                             |
| id        | 0fda2105345d4f1ca5c1f32c7118bf25 |
| name      | glance                           |
+-----------+----------------------------------+
========<
```


-  ロールの割り当て [対象: controller01]

adminロールを glanceユーザーと serviceプロジェクト に追加する

```
# openstack role add --project service --user glance admin
```


- glanceサービスエンティティー の作成 [対象: controller01]

```
# openstack service create --name glance --description "OpenStack Image service" image
========>
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image service          |
| enabled     | True                             |
| id          | 8d6812e182e64c13a7149e9181c1509e |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
========<
```

### APIエンドポイント の作成

-  public, internal, admin 用の APIエンドポイント の作成 [対象: controller01]

```
# openstack endpoint create --region RegionOne image public http://controller01:9292
========>
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 24ea67411d524130a35cc1d5c583383b |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8d6812e182e64c13a7149e9181c1509e |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller01:9292         |
+--------------+----------------------------------+
========<


# openstack endpoint create --region RegionOne image internal http://controller01:9292
========>
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | c852c160ac484c47843b1e636b0878f5 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8d6812e182e64c13a7149e9181c1509e |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller01:9292         |
+--------------+----------------------------------+
========<


# openstack endpoint create --region RegionOne image admin http://controller:9292
========>
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 7066883e023e491c896697a4441a0344 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8d6812e182e64c13a7149e9181c1509e |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
========<
```


## コンポーネントのインストール と 設定

### パッケージのインストール

- インストール [対象: controller01]

```
# yum install -y openstack-glance python-glance python-glanceclient
========>
Installed:
  openstack-glance.noarch 1:11.0.1-3.el7                                  python-glance.noarch 1:11.0.1-3.el7

Dependency Installed:
  atlas.x86_64 0:3.10.1-10.el7                       blas.x86_64 0:3.4.2-5.el7                    crudini.noarch 0:0.7-1.el7
  lapack.x86_64 0:3.4.2-5.el7                        libgfortran.x86_64 0:4.8.5-4.el7             libquadmath.x86_64 0:4.8.5-4.el7
  numpy.x86_64 1:1.7.1-11.el7                        numpy-f2py.x86_64 1:1.7.1-11.el7             openstack-utils.noarch 0:2015.2-1.el7
  pysendfile.x86_64 0:2.0.0-5.el7                    python-automaton.noarch 0:0.7.0-1.el7        python-boto.noarch 0:2.34.0-4.el7
  python-devel.x86_64 0:2.7.5-34.el7                 python-glance-store.noarch 0:0.9.1-1.el7     python-networkx-core.noarch 0:1.10-1.el7
  python-nose.noarch 0:1.3.7-7.el7                   python-oslo-vmware.noarch 0:1.21.0-1.el7     python-osprofiler.noarch 0:0.3.0-1.el7
  python-semantic_version.noarch 0:2.4.2-1.el7       python-simplegeneric.noarch 0:0.8-7.el7      python-swiftclient.noarch 0:2.6.0-1.el7
  python-taskflow.noarch 0:1.21.0-1.el7              python2-castellan.noarch 0:0.3.1-1.el7       python2-rsa.noarch 0:3.3-2.el7
  python2-suds.noarch 0:0.7-0.1.94664ddd46a6.el7     python2-wsme.noarch 0:0.7.0-2.el7            scipy.x86_64 0:0.12.1-3.el7
  suitesparse.x86_64 0:4.0.2-10.el7                  tbb.x86_64 0:4.1-9.20130314.el7

Complete!
========<
```


### 設定

- 設定 glance-api.conf [対象: controller01]

```
# vi /etc/glance/glance-api.conf
========> 以下を参考に編集
[database]
connection = mysql://glance:Password123$@controller/glance

[keystone_authtoken]
auth_uri = http://controller01:5000
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = glance
password = Password123$
# <[keystone_authtoken] に他の項目があった場合、全てコメントアウトする>

[paste_deploy]
flavor = keystone

[glance_store]
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

[DEFAULT]
notification_driver = noop

# <以下オプション>
# <今回は設定しない>
[DEFAULT]
verbose = True
========<
```


- 設定 glance-registry.conf [対象: controller01]

```
# vi /etc/glance/glance-registry.conf
========> 以下を参考に編集
[database]
connection = mysql://glance:Password123$@controller01/glance

[keystone_authtoken]
auth_uri = http://controller01:5000
auth_url = http://controller01:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = glance
password = Password123$
# <[keystone_authtoken] に他の項目があった場合、全てコメントアウトする>

[paste_deploy]
flavor = keystone

[DEFAULT]
notification_driver = noop

# <以下オプション>
# <今回は設定しない>
[DEFAULT]
verbose = True
========<
```


## データベースへの展開

- データベースへの展開 [対象: controller01]

```
# su -s /bin/sh -c "glance-manage db_sync" glance
========> 以下、無視！
No handlers could be found for logger "oslo_config.cfg"
========<
```

## Imageサービスを自動起動設定 と 起動

- 自動起動設定 と 起動 [対象: controller01]

```
# systemctl enable openstack-glance-api.service openstack-glance-registry.service
========>
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-glance-api.service to /usr/lib/systemd/system/openstack-glance-api.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-glance-registry.service to /usr/lib/systemd/system/openstack-glance-registry.service.
========<


# systemctl start openstack-glance-api.service openstack-glance-registry.service
```


## データベースへの展開 確認

- 確認 [対象: controller01]

```
[root@controller01 ~]# mysql -u glance -h controller01 -p
Enter password: Password123$  <== 入力中は表示されない
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 23
Server version: 5.5.50-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| glance             |
+--------------------+
2 rows in set (0.00 sec)

MariaDB [(none)]>
MariaDB [(none)]> USE glance;
Database changed
MariaDB [glance]>
MariaDB [glance]> SHOW TABLES;
Empty set (0.00 sec)

ありゃ？
空でいいんだっけ？？？
あぁ、、、、何も登録してないか、、、、、。
```



## 動作確認

- 環境スクリプト追加 [対象: controller01]

API version 2.0 を使用するよう Image service クライアントを設定

```
# echo "export OS_IMAGE_API_VERSION=2" | tee -a ~/admin-openrc.sh ~/demo-openrc.sh
========>
export OS_IMAGE_API_VERSION=2
========<
```

- 環境スクリプト(adminクレデンシャル) の読み込み [対象: controller01]

管理者専用CLIコマンドへのアクセス権を読み込む

```
# source admin-openrc.sh
```

- 動作確認用OSのダウンロード

```
# yum install -y wget
========>
Installed:
  wget.x86_64 0:1.14-10.el7_0.1

Complete!
========<

# wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
```


- イメージのアップロード

ダウンロードしたイメージを、 QCOW2 ディスク形式、bare コンテナー形式、パブリック公開で Image service にアップロードする。パブリック公開を指定すると、すべてのプロジェクトがこのイメージにアクセス出来る。

```
# glance image-create --name "cirros" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --visibility public --progress

========>
500 Internal Server Error: The server has either erred or is incapable of performing the requested operation. (HTTP 500)
========<

ありゃ？？？？

```
