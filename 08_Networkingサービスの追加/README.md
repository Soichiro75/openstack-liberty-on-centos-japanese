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

「オプション2 セルフサービスネットワーク」 にて構築する


### パッケージのインストール (コントローラーノード)

- インストール [対象: controller01]
