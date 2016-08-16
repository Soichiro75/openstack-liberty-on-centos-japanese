# Identityサービスの追加

http://docs.openstack.org/liberty/ja/install-guide-rdo/keystone.html

OpenStack Identity サービス(Keystone) は、認証、認可、サービスカタログサービスを管理する OpenStack のサービス。他の OpenStack サービスは共通の統一 API として Identity サービスを使用出来る。

この手順では、コントローラーノードに OpenStack Identity サービス (コード名 keystone) をインストールして設定する。今回は、パフォーマンスを重視し、Apache HTTP サーバーを使ってリクエストを処理し、SQL データベースの代わりに Memcached を使ってトークンを保存する。

## Identityサービス の インストールと設定

### 事前準備  データベース と 管理トークン 作成

- ログイン [対象: controller01]

```
# mysql -u root -p
========>
Enter password: Password123$    <=== 入力中は表示されない
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 2
Server version: 5.5.50-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
========<
```


- keystone データベースの作成 [対象: controller01]

```
MariaDB [(none)]> SHOW DATABASES;
========>
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
3 rows in set (0.02 sec)
========<


MariaDB [(none)]> CREATE DATABASE keystone;
========>
Query OK, 1 row affected (0.00 sec)
========<


MariaDB [(none)]> SHOW DATABASES;
========>
+--------------------+
| Database           |
+--------------------+
| information_schema |
| keystone           |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.00 sec)
========<
```


- keystoneデータベースへのアクセス権設定 [対象: controller01]
  - 補足：
    - GRANT(権限設定) ALL PRIVILEGES(全ての権限) ON(付与) [データベース名].[テーブル名]\(対象\) [ユーザー]@[ホスト(%:ワイルドカード)]\(ユーザー名と接続元ホスト名\) IDENTIFIED BY(ユーザーのパスワード)

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'Password123$';
========>
Query OK, 0 rows affected (0.00 sec)
========<


MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'Password123$';
========>
Query OK, 0 rows affected (0.00 sec)
========<


MariaDB [(none)]> SHOW GRANTS FOR keystone;
========>
+---------------------------------------------------------------------------------------------------------+
| Grants for keystone@%                                                                                   |
+---------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'keystone'@'%' IDENTIFIED BY PASSWORD '*6C12FC2B651683A59ADD31A5C49E1E6F801D399F' |
| GRANT ALL PRIVILEGES ON `keystone`.* TO 'keystone'@'%'                                                  |
+---------------------------------------------------------------------------------------------------------+
2 rows in set (0.01 sec)
========<

MariaDB [(none)]> exit
========>
Bye
[root@controller01 ~]#
========<
```


- 初期設定用トークン作成 [対象: controller01]

```
# <ここでは、ランダムな6真数値の代わりにPassword123$とする>
# openssl rand -hex 10
========>
Password123$
========<
```


### コンポーネントのインストール と 設定

(注釈) Kilo リリースと Liberty リリースでは、keystone プロジェクトは eventlet を非推奨扱い。代わりに WSGI 拡張に対応した専用 Web サーバーの使用を推奨。この手順では、Apache HTTP server の mod_wsgi を使用して、5000 番ポートと 35357 番ポートで Identity サービスのリクエストを処理する。デフォルトでは、keystone サービスは、 5000 番と 35357 番をリッスンしている。そのため、この手順では、keystone サービスを無効化する。keystone プロジェクトは、Mitaka リリースで eventlet のサポートを削除する予定。
<!--
keystone の無効化ってどこ？
-->


補足：
 mod_wsgiとは、WSGI (Web Server Gateway Interface) インターフェースに準拠した PythonのプログラムをApache HTTP Serverで動作させるためのモジュールである。


- コンポーネントのインストール [対象: controller01]

```
# yum install -y openstack-keystone httpd mod_wsgi memcached python-memcached
========>
Installed:
  httpd.x86_64 0:2.4.6-40.el7.centos.4 memcached.x86_64 0:1.4.25-1.el7 mod_wsgi.x86_64 0:3.4-12.el7_0 openstack-keystone.noarch 1:8.1.2-1.el7
  python-memcached.noarch 0:1.54-3.el7

Dependency Installed:
  PyPAM.x86_64 0:0.5.0-19.el7                                       apr.x86_64 0:1.4.8-3.el7
  apr-util.x86_64 0:1.5.2-6.el7                                     httpd-tools.x86_64 0:2.4.6-40.el7.centos.4
  libevent.x86_64 0:2.0.21-4.el7                                    mailcap.noarch 0:2.1.41-2.el7
  python-alembic.noarch 0:0.8.3-3.el7                               python-amqp.noarch 0:1.4.6-1.el7
  python-anyjson.noarch 0:0.3.3-3.el7                               python-beaker.noarch 0:1.5.4-10.el7
  python-cachetools.noarch 0:1.0.3-2.el7                            python-contextlib2.noarch 0:0.4.0-1.el7
  python-dateutil.noarch 0:1.5-7.el7                                python-dogpile-cache.noarch 0:0.5.7-3.el7
  python-dogpile-core.noarch 0:0.4.1-2.el7                          python-editor.noarch 0:0.4-4.el7
  python-futures.noarch 0:3.0.3-1.el7                               python-keystone.noarch 1:8.1.2-1.el7
  python-keystonemiddleware.noarch 0:2.3.1-1.el7                    python-kombu.noarch 1:3.0.32-1.el7
  python-ldap.x86_64 0:2.4.15-2.el7                                 python-ldappool.noarch 0:1.0-4.el7
  python-mako.noarch 0:0.8.1-2.el7                                  python-markupsafe.x86_64 0:0.11-10.el7
  python-migrate.noarch 0:0.10.0-1.el7                              python-oauthlib.noarch 0:0.7.2-5.20150520git514cad7.el7
  python-oslo-concurrency.noarch 0:2.6.0-1.el7                      python-oslo-db.noarch 0:2.6.0-3.el7
  python-oslo-log.noarch 0:1.10.0-1.el7                             python-oslo-messaging.noarch 0:2.5.0-1.el7
  python-oslo-middleware.noarch 0:2.8.0-1.el7                       python-oslo-policy.noarch 0:0.11.0-1.el7
  python-oslo-service.noarch 0:0.9.0-1.el7                          python-paste.noarch 0:1.7.5.1-9.20111221hg1498.el7
  python-paste-deploy.noarch 0:1.5.2-6.el7                          python-posix_ipc.x86_64 0:0.9.8-1.el7
  python-pycadf.noarch 0:1.1.0-1.el7                                python-pysaml2.noarch 0:3.0.2-1.el7
  python-qpid.noarch 0:0.32-9.el7                                   python-qpid-common.noarch 0:0.32-9.el7
  python-repoze-lru.noarch 0:0.4-3.el7                              python-repoze-who.noarch 0:2.1-1.el7
  python-retrying.noarch 0:1.2.3-4.el7                              python-routes.noarch 0:1.13-2.el7
  python-saslwrapper.x86_64 0:0.16-5.el7                            python-sqlalchemy.x86_64 0:1.0.11-1.el7
  python-sqlparse.noarch 0:0.1.18-5.el7                             python-tempita.noarch 0:0.5.1-8.el7
  python-zope-interface.x86_64 0:4.0.5-4.el7                        python2-PyMySQL.noarch 0:0.6.7-2.el7
  python2-eventlet.noarch 0:0.17.4-4.el7                            python2-fasteners.noarch 0:0.14.1-4.el7
  python2-futurist.noarch 0:0.5.0-1.el7                             python2-greenlet.x86_64 0:0.4.9-1.el7
  python2-oslo-context.noarch 0:0.6.0-1.el7                         python2-passlib.noarch 0:1.6.5-1.el7
  saslwrapper.x86_64 0:0.16-5.el7

Complete!
========<
```


- memcached の自動起動設定と、起動 [対象: controller01]

```
# systemctl enable memcached.service
========>
Created symlink from /etc/systemd/system/multi-user.target.wants/memcached.service to /usr/lib/systemd/system/memcached.service.
========<


# systemctl start memcached.service
```


- keystone の設定 [対象: controller01]
  - 補足：
    - [DEFAULT] セクションに初期管理トークンの値を定義
    - [database] セクションで、データベースのアクセス方法を設定
    - [memcache] セクションで Memcached サービスを設定
    - [token] セクションで UUID トークンプロバイダーと memcached ドライバーを設定
    - [revoke] セクションで SQL revocation ドライバーを設定
    - (オプション) トラブルシューティングしやすくするために、冗長ロギングを [DEFAULT] セクションで有効化


```
# vi /etc/keystone/keystone.conf
========>以下を参考に、編集 or 追加
[DEFAULT]
admin_token = Password123$

[database]
connection = mysql://keystone:Password123$@controller01/keystone

[memcache]
servers = localhost:11211

[token]
provider = uuid
driver = memcache

[revoke]
driver = sql

### <以下はオプション>
### <この手順では追加しない>
### <追加すると、keystone-manage db_sync の時に、>
### <No handlers could be found for logger "oslo_config.cfg" ってエラーが出るため>
[DEFAULT]
verbose = true
========<
```


- Identyty サービス(keystone)データベースの展開 [対象: controller01]

```
# su -s /bin/sh -c "keystone-manage db_sync" keystone
```


- データベース確認 [対象: controller01]

```
# mysql -u keystone -h controller01 -p
========>
Enter password: Password123$
========<


MariaDB [(none)]> SHOW DATABASES;
========>
+--------------------+
| Database           |
+--------------------+
| information_schema |
| keystone           |
+--------------------+
2 rows in set (0.00 sec)
========<


MariaDB [(none)]> USE keystone;


MariaDB [keystone]> SHOW TABLES;
========>
+------------------------+
| Tables_in_keystone     |
+------------------------+
| access_token           |
| assignment             |
| config_register        |
| consumer               |
| credential             |
| domain                 |
| endpoint               |
| endpoint_group         |
| federation_protocol    |
| group                  |
| id_mapping             |
| identity_provider      |
| idp_remote_ids         |
| mapping                |
| migrate_version        |
| policy                 |
| policy_association     |
| project                |
| project_endpoint       |
| project_endpoint_group |
| region                 |
| request_token          |
| revocation_event       |
| role                   |
| sensitive_config       |
| service                |
| service_provider       |
| token                  |
| trust                  |
| trust_role             |
| user                   |
| user_group_membership  |
| whitelisted_config     |
+------------------------+
33 rows in set (0.00 sec)
========<

MariaDB [keystone]>exit
```

<!--
```
[root@controller01 ~]# keystone-manage db_sync keystone
========>
No handlers could be found for logger "oslo_config.cfg"
========<
↑ コマンド、違う。 su でkeystoneで実行すべき


su -s /bin/sh -c "keystone-manage db_sync" keystone
========>
No handlers could be found for logger "oslo_config.cfg"
Traceback (most recent call last):
  File "/usr/bin/keystone-manage", line 10, in <module>
    sys.exit(main())
  File "/usr/lib/python2.7/site-packages/keystone/cmd/manage.py", line 46, in main
    cli.main(argv=sys.argv, config_files=config_files)
  File "/usr/lib/python2.7/site-packages/keystone/cmd/cli.py", line 684, in main
    config.setup_logging()
  File "/usr/lib/python2.7/site-packages/keystone/config.py", line 56, in setup_logging
    log.setup(CONF, 'keystone')
  File "/usr/lib/python2.7/site-packages/oslo_log/log.py", line 246, in setup
    _setup_logging_from_conf(conf, product_name, version)
  File "/usr/lib/python2.7/site-packages/oslo_log/log.py", line 314, in _setup_logging_from_conf
    filelog = logging.handlers.WatchedFileHandler(logpath)
  File "/usr/lib64/python2.7/logging/handlers.py", line 392, in __init__
    logging.FileHandler.__init__(self, filename, mode, encoding, delay)
  File "/usr/lib64/python2.7/logging/__init__.py", line 902, in __init__
    StreamHandler.__init__(self, self._open())
  File "/usr/lib64/python2.7/logging/__init__.py", line 925, in _open
    stream = open(self.baseFilename, self.mode)
IOError: [Errno 13] Permission denied: '/var/log/keystone/keystone.log'
========<

↑これは、この前に一度rootで実行して、root権限になったから？
  → その通り。以下、keystone.logを削除して、keystoneユーザーでDBsyncし直したら(ログ作成)、上記 Permission deniedエラーは出なくなった。


[root@controller01 ~]# ls -ld /var/log/keystone/
drwxr-x---. 2 keystone keystone 25 Aug 15 17:31 /var/log/keystone/

[root@controller01 ~]# ls -ld /var/log/keystone/*
-rw-r--r--. 1 root root 8881 Aug 15 17:34 /var/log/keystone/keystone.log

[root@controller01 ~]# mv /var/log/keystone/keystone.log /tmp

[root@controller01 ~]# ls -ld /var/log/keystone/keystone.log
ls: cannot access /var/log/keystone/keystone.log: No such file or directory

[root@controller01 ~]# su -s /bin/sh -c "keystone-manage db_sync" keystone
No handlers could be found for logger "oslo_config.cfg"

[root@controller01 ~]# ls -ld /var/log/keystone/keystone.log
-rw-r--r--. 1 keystone keystone 0 Aug 16 09:36 /var/log/keystone/keystone.log



No handlers could be found for logger "oslo_config.cfg"
ってなに？？？？？？？？？？？？



[root@controller01 ~]# mysql -u keystone -h controller01 -p
Enter password: Password123$

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| keystone           |
+--------------------+
2 rows in set (0.00 sec)

MariaDB [(none)]>


データベースは出来ているみたい。

MariaDB [(none)]> use keystone;

MariaDB [keystone]> show tables;
+------------------------+
| Tables_in_keystone     |
+------------------------+
| access_token           |
| assignment             |
| config_register        |
| consumer               |
| credential             |
| domain                 |
| endpoint               |
| endpoint_group         |
| federation_protocol    |
| group                  |
| id_mapping             |
| identity_provider      |
| idp_remote_ids         |
| mapping                |
| migrate_version        |
| policy                 |
| policy_association     |
| project                |
| project_endpoint       |
| project_endpoint_group |
| region                 |
| request_token          |
| revocation_event       |
| role                   |
| sensitive_config       |
| service                |
| service_provider       |
| token                  |
| trust                  |
| trust_role             |
| user                   |
| user_group_membership  |
| whitelisted_config     |
+------------------------+
33 rows in set (0.00 sec)

MariaDB [keystone]>


見つけた！
https://ask.openstack.org/en/question/86471/no-handlers-could-be-found-for-logger-oslo_configcfg/
"verbose" の設定を  "/etc/keystone/keystone.conf" から削除してから、
OS reboot したら、No handlers could be found for logger "oslo_config.cfg" 出なくなった。
```
-->

### Apache HTTP Server の設定

- Apacheのホスト名設定(ServerName) [対象: controller01]

```
vi /etc/httpd/conf/httpd.conf
========>以下を参考に 編集
ServerName controller01
========<
```

- mod_wsgiの設定 [対象: controller01]

  - 補足：
    - mod_wsgi: PythonをApache HTTP Serverで動作させるためのモジュール
    - /etc/httpd/conf.d/ :Apacheのモジュールの設定ファイル置き場

```
vi /etc/httpd/conf.d/wsgi-keystone.conf
========>以下を参考に 新規作成 (特に変更箇所はないはず)
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/httpd/keystone-error.log
    CustomLog /var/log/httpd/keystone-access.log combined

    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/httpd/keystone-error.log
    CustomLog /var/log/httpd/keystone-access.log combined

    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>
========<
```


- Apache の自動起動設定と、起動 [対象: controller01]

```
# systemctl enable httpd.service
========>
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
========<


# systemctl start httpd.service
```



## サービスエンティティと API エンドポイントの作成

Identity サービスのデータベースは、初期状態では、認証やカタログサービスの情報を何も持っていない。インストールと設定 セクションにおいて作成した一時認証トークン(openssl rand -hex 10)を使用して、Identity サービスのサービスエンティティーと API エンドポイントを初期化する必要がある。

openstack コマンドの --os-token パラメーターに認証トークンの値を渡すか、OS_TOKEN 環境変数を設定する必要がある。同様に、openstack コマンドの --os-url パラメーターに Identity サービスの URL 値を渡すか、OS_URL 環境変数を設定する必要がある。この手順では、コマンドの長さを短くするため、環境変数を使用する。

### 環境変数設定

- 環境変数設定 [対象: controller01]

export コマンドを直接設定しても良いが、環境構築中にOS再起動した時に、再設定を忘れそうなので、`.bashrc`に記述する

`export OS_TOKEN=Password123$`: 認証トークン、本手順の上部で生成した`openssl rand -hex 10`の値

`export OS_URL=http://controller01:35357/v3`: エンドポイントURL

`export OS_IDENTITY_API_VERSION=3`: Identity APIバージョン


```
# vi ~/.bashrc
========>以下を追加
export OS_TOKEN=Password123$
export OS_URL=http://controller01:35357/v3
export OS_IDENTITY_API_VERSION=3
========<


# source ~/.bashrc
```


### Identityサービス用 サービスエンティティ作成

<!--
### 自分用メモ

この後、keystoneのDBに登録していくはず、そのため、事前確認しておきたい

- 事前確認 [対象: controller01]
```
# [root@controller01 ~]# mysql -u keystone -h controller01 -p

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| keystone           |
+--------------------+
2 rows in set (0.00 sec)

MariaDB [(none)]> use keystone;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed



MariaDB [keystone]> show tables;
+------------------------+
| Tables_in_keystone     |
+------------------------+
| access_token           |
| assignment             |
| config_register        |
| consumer               |
| credential             |
| domain                 |
| endpoint               |
| endpoint_group         |
| federation_protocol    |
| group                  |
| id_mapping             |
| identity_provider      |
| idp_remote_ids         |
| mapping                |
| migrate_version        |
| policy                 |
| policy_association     |
| project                |
| project_endpoint       |
| project_endpoint_group |
| region                 |
| request_token          |
| revocation_event       |
| role                   |
| sensitive_config       |
| service                |
| service_provider       |
| token                  |
| trust                  |
| trust_role             |
| user                   |
| user_group_membership  |
| whitelisted_config     |
+------------------------+
33 rows in set (0.00 sec)





MariaDB [keystone]> select * from access_token;
Empty set (0.00 sec)

MariaDB [keystone]> select * from assignment;
Empty set (0.00 sec)

MariaDB [keystone]> select * from config_register;
Empty set (0.00 sec)

MariaDB [keystone]> select * from consumer;
Empty set (0.00 sec)

MariaDB [keystone]> select * from credential;
Empty set (0.00 sec)

MariaDB [keystone]> select * from domain;
+---------+---------+---------+-----------------------------------------------------------------------------------------+
| id      | name    | enabled | extra                                                                                   |
+---------+---------+---------+-----------------------------------------------------------------------------------------+
| default | Default |       1 | {"description": "Owns users and tenants (i.e. projects) available on Identity API v2."} |
+---------+---------+---------+-----------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

MariaDB [keystone]> select * from endpoint;
Empty set (0.00 sec)

MariaDB [keystone]> select * from endpoint_group;
Empty set (0.00 sec)

MariaDB [keystone]> select * from federation_protocol;
Empty set (0.00 sec)

MariaDB [keystone]> select * from group;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near 'group' at line 1
MariaDB [keystone]> select * from id_mapping;
Empty set (0.00 sec)

MariaDB [keystone]> select * from identity_provider;
Empty set (0.00 sec)

MariaDB [keystone]> select * from idp_remote_ids;
Empty set (0.00 sec)

MariaDB [keystone]> select * from mapping;
Empty set (0.00 sec)

MariaDB [keystone]> select * from migrate_version;
+-----------------+--------------------------------------------------------------------------------+---------+
| repository_id   | repository_path                                                                | version |
+-----------------+--------------------------------------------------------------------------------+---------+
| endpoint_filter | /usr/lib/python2.7/site-packages/keystone/contrib/endpoint_filter/migrate_repo |       2 |
| endpoint_policy | /usr/lib/python2.7/site-packages/keystone/contrib/endpoint_policy/migrate_repo |       1 |
| federation      | /usr/lib/python2.7/site-packages/keystone/contrib/federation/migrate_repo      |       8 |
| keystone        | /usr/lib/python2.7/site-packages/keystone/common/sql/migrate_repo              |      75 |
| oauth1          | /usr/lib/python2.7/site-packages/keystone/contrib/oauth1/migrate_repo          |       5 |
| revoke          | /usr/lib/python2.7/site-packages/keystone/contrib/revoke/migrate_repo          |       2 |
+-----------------+--------------------------------------------------------------------------------+---------+
6 rows in set (0.00 sec)

MariaDB [keystone]> select * from policy;
Empty set (0.00 sec)

MariaDB [keystone]> select * from policy_association;
Empty set (0.00 sec)

MariaDB [keystone]> select * from project;
Empty set (0.00 sec)

MariaDB [keystone]> select * from project_endpoint;
Empty set (0.00 sec)

MariaDB [keystone]> select * from project_endpoint_group;
Empty set (0.00 sec)

MariaDB [keystone]> select * from region;
Empty set (0.00 sec)

MariaDB [keystone]> select * from request_token;
Empty set (0.00 sec)

MariaDB [keystone]> select * from revocation_event;
Empty set (0.00 sec)

MariaDB [keystone]> select * from role;
Empty set (0.00 sec)

MariaDB [keystone]> select * from sensitive_config;
Empty set (0.00 sec)

MariaDB [keystone]> select * from service;
Empty set (0.00 sec)

MariaDB [keystone]> select * from service_provider;
Empty set (0.00 sec)

MariaDB [keystone]> select * from token;
Empty set (0.00 sec)

MariaDB [keystone]> select * from trust;
Empty set (0.00 sec)

MariaDB [keystone]> select * from trust_role;
Empty set (0.00 sec)

MariaDB [keystone]> select * from user;
Empty set (0.00 sec)

MariaDB [keystone]> select * from user_group_membership;
Empty set (0.00 sec)

MariaDB [keystone]> select * from whitelisted_config;
Empty set (0.00 sec)

MariaDB [keystone]>
```
-->


- Identityサービス用 の サービスエンティティ の作成 [対象: controller01]

```
# openstack service create --name keystone --description "OpenStack Identity" identity
========>
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Identity               |
| enabled     | True                             |
| id          | 20ac0030158744a499e0c9b04ba077a5 |
| name        | keystone                         |
| type        | identity                         |
+-------------+----------------------------------+
========<
```

<!--
自分メモ
たしかに、serviceにidentityが追加されている。

MariaDB [keystone]> select * from access_token;
Empty set (0.00 sec)

MariaDB [keystone]> select * from assignment;
Empty set (0.00 sec)

MariaDB [keystone]> select * from config_register;
Empty set (0.00 sec)

MariaDB [keystone]> select * from consumer;
Empty set (0.00 sec)

MariaDB [keystone]> select * from credential;
Empty set (0.00 sec)

MariaDB [keystone]> select * from domain;
+---------+---------+---------+-----------------------------------------------------------------------------------------+
| id      | name    | enabled | extra                                                                                   |
+---------+---------+---------+-----------------------------------------------------------------------------------------+
| default | Default |       1 | {"description": "Owns users and tenants (i.e. projects) available on Identity API v2."} |
+---------+---------+---------+-----------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

MariaDB [keystone]> select * from endpoint;
Empty set (0.00 sec)

MariaDB [keystone]> select * from endpoint_group;
Empty set (0.00 sec)

MariaDB [keystone]> select * from federation_protocol;
Empty set (0.00 sec)

MariaDB [keystone]> select * from group;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near 'group' at line 1
MariaDB [keystone]> select * from id_mapping;
Empty set (0.00 sec)

MariaDB [keystone]> select * from identity_provider;
Empty set (0.00 sec)

MariaDB [keystone]> select * from idp_remote_ids;
Empty set (0.00 sec)

MariaDB [keystone]> select * from mapping;
Empty set (0.00 sec)

MariaDB [keystone]> select * from migrate_version;
+-----------------+--------------------------------------------------------------------------------+---------+
| repository_id   | repository_path                                                                | version |
+-----------------+--------------------------------------------------------------------------------+---------+
| endpoint_filter | /usr/lib/python2.7/site-packages/keystone/contrib/endpoint_filter/migrate_repo |       2 |
| endpoint_policy | /usr/lib/python2.7/site-packages/keystone/contrib/endpoint_policy/migrate_repo |       1 |
| federation      | /usr/lib/python2.7/site-packages/keystone/contrib/federation/migrate_repo      |       8 |
| keystone        | /usr/lib/python2.7/site-packages/keystone/common/sql/migrate_repo              |      75 |
| oauth1          | /usr/lib/python2.7/site-packages/keystone/contrib/oauth1/migrate_repo          |       5 |
| revoke          | /usr/lib/python2.7/site-packages/keystone/contrib/revoke/migrate_repo          |       2 |
+-----------------+--------------------------------------------------------------------------------+---------+
6 rows in set (0.00 sec)

MariaDB [keystone]> select * from policy;
Empty set (0.00 sec)

MariaDB [keystone]> select * from policy_association;
Empty set (0.00 sec)

MariaDB [keystone]> select * from project;
Empty set (0.00 sec)

MariaDB [keystone]> select * from project_endpoint;
Empty set (0.00 sec)

MariaDB [keystone]> select * from project_endpoint_group;
Empty set (0.00 sec)

MariaDB [keystone]> select * from region;
Empty set (0.00 sec)

MariaDB [keystone]> select * from request_token;
Empty set (0.00 sec)

MariaDB [keystone]> select * from revocation_event;
Empty set (0.00 sec)

MariaDB [keystone]> select * from role;
Empty set (0.00 sec)

MariaDB [keystone]> select * from sensitive_config;
Empty set (0.00 sec)

MariaDB [keystone]> select * from service;
+----------------------------------+----------+---------+-----------------------------------------------------------+
| id                               | type     | enabled | extra                                                     |
+----------------------------------+----------+---------+-----------------------------------------------------------+
| 20ac0030158744a499e0c9b04ba077a5 | identity |       1 | {"description": "OpenStack Identity", "name": "keystone"} |
+----------------------------------+----------+---------+-----------------------------------------------------------+
1 row in set (0.00 sec)

MariaDB [keystone]> select * from service_provider;
Empty set (0.00 sec)

MariaDB [keystone]> select * from token;
Empty set (0.00 sec)

MariaDB [keystone]> select * from trust;
Empty set (0.00 sec)

MariaDB [keystone]> select * from trust_role;
Empty set (0.00 sec)

MariaDB [keystone]> select * from user;
Empty set (0.00 sec)

MariaDB [keystone]> select * from user_group_membership;
Empty set (0.00 sec)

MariaDB [keystone]> select * from whitelisted_config;
Empty set (0.00 sec)

MariaDB [keystone]>

-->

- Identityサービス APIエンドポイント の作成 [対象: controller01]

  - 補足
    - ResionOne: デフォルトのリージョン

```
openstack endpoint create --region RegionOne identity public http://controller01:5000/v2.0
========>
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | ca4c49ed7b934ba489ad58540576e290 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 20ac0030158744a499e0c9b04ba077a5 |
| service_name | keystone                         |
| service_type | identity                         |
| url          | http://controller01:5000/v2.0    |
+--------------+----------------------------------+
========<


openstack endpoint create --region RegionOne identity internal http://controller01:5000/v2.0
========>
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 192449e045694288a80d6dd98037b3dc |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 20ac0030158744a499e0c9b04ba077a5 |
| service_name | keystone                         |
| service_type | identity                         |
| url          | http://controller01:5000/v2.0    |
+--------------+----------------------------------+
========<


openstack endpoint create --region RegionOne identity admin http://controller01:35357/v2.0
========>
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 44ee326d2e2d473f8613e063f913ab38 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 20ac0030158744a499e0c9b04ba077a5 |
| service_name | keystone                         |
| service_type | identity                         |
| url          | http://controller01:35357/v2.0   |
+--------------+----------------------------------+
========<
```


- データベースへの登録確認 [対象: controller01]

気になったからテーブル参照しただけ、以下の確認コマンドの実行は必要ないので、飛ばして次に進んでOK


```
[root@controller01 ~]# mysql -u keystone -h controller01 -p
Enter password: Password123$

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 11
Server version: 5.5.50-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| keystone           |
+--------------------+
2 rows in set (0.00 sec)

MariaDB [(none)]>
MariaDB [(none)]> USE keystone;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [keystone]>
MariaDB [keystone]> SHOW TABLES;
+------------------------+
| Tables_in_keystone     |
+------------------------+
| access_token           |
| assignment             |
| config_register        |
| consumer               |
| credential             |
| domain                 |
| endpoint               |
| endpoint_group         |
| federation_protocol    |
| group                  |
| id_mapping             |
| identity_provider      |
| idp_remote_ids         |
| mapping                |
| migrate_version        |
| policy                 |
| policy_association     |
| project                |
| project_endpoint       |
| project_endpoint_group |
| region                 |
| request_token          |
| revocation_event       |
| role                   |
| sensitive_config       |
| service                |
| service_provider       |
| token                  |
| trust                  |
| trust_role             |
| user                   |
| user_group_membership  |
| whitelisted_config     |
+------------------------+
33 rows in set (0.00 sec)

MariaDB [keystone]>

MariaDB [keystone]> select * from access_token;
Empty set (0.00 sec)

MariaDB [keystone]> select * from assignment;
Empty set (0.00 sec)

MariaDB [keystone]> select * from config_register;
Empty set (0.00 sec)

MariaDB [keystone]> select * from consumer;
Empty set (0.00 sec)

MariaDB [keystone]> select * from credential;
Empty set (0.00 sec)

MariaDB [keystone]> select * from domain;
+---------+---------+---------+-----------------------------------------------------------------------------------------+
| id      | name    | enabled | extra                                                                                   |
+---------+---------+---------+-----------------------------------------------------------------------------------------+
| default | Default |       1 | {"description": "Owns users and tenants (i.e. projects) available on Identity API v2."} |
+---------+---------+---------+-----------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

MariaDB [keystone]> select * from endpoint;
+----------------------------------+--------------------+-----------+----------------------------------+--------------------------------+-------+---------+-----------+
| id                               | legacy_endpoint_id | interface | service_id                       | url                            | extra | enabled | region_id |
+----------------------------------+--------------------+-----------+----------------------------------+--------------------------------+-------+---------+-----------+
| 192449e045694288a80d6dd98037b3dc | NULL               | internal  | 20ac0030158744a499e0c9b04ba077a5 | http://controller01:5000/v2.0  | {}    |       1 | RegionOne |
| 44ee326d2e2d473f8613e063f913ab38 | NULL               | admin     | 20ac0030158744a499e0c9b04ba077a5 | http://controller01:35357/v2.0 | {}    |       1 | RegionOne |
| ca4c49ed7b934ba489ad58540576e290 | NULL               | public    | 20ac0030158744a499e0c9b04ba077a5 | http://controller01:5000/v2.0  | {}    |       1 | RegionOne |
+----------------------------------+--------------------+-----------+----------------------------------+--------------------------------+-------+---------+-----------+
3 rows in set (0.00 sec)

MariaDB [keystone]> select * from endpoint_group;
Empty set (0.00 sec)

MariaDB [keystone]> select * from federation_protocol;
Empty set (0.00 sec)

MariaDB [keystone]> select * from group;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near 'group' at line 1
MariaDB [keystone]> select * from id_mapping;
Empty set (0.00 sec)

MariaDB [keystone]> select * from identity_provider;
Empty set (0.01 sec)

MariaDB [keystone]> select * from idp_remote_ids;
Empty set (0.00 sec)

MariaDB [keystone]> select * from mapping;
Empty set (0.00 sec)

MariaDB [keystone]> select * from migrate_version;
+-----------------+--------------------------------------------------------------------------------+---------+
| repository_id   | repository_path                                                                | version |
+-----------------+--------------------------------------------------------------------------------+---------+
| endpoint_filter | /usr/lib/python2.7/site-packages/keystone/contrib/endpoint_filter/migrate_repo |       2 |
| endpoint_policy | /usr/lib/python2.7/site-packages/keystone/contrib/endpoint_policy/migrate_repo |       1 |
| federation      | /usr/lib/python2.7/site-packages/keystone/contrib/federation/migrate_repo      |       8 |
| keystone        | /usr/lib/python2.7/site-packages/keystone/common/sql/migrate_repo              |      75 |
| oauth1          | /usr/lib/python2.7/site-packages/keystone/contrib/oauth1/migrate_repo          |       5 |
| revoke          | /usr/lib/python2.7/site-packages/keystone/contrib/revoke/migrate_repo          |       2 |
+-----------------+--------------------------------------------------------------------------------+---------+
6 rows in set (0.00 sec)

MariaDB [keystone]> select * from policy;
Empty set (0.00 sec)

MariaDB [keystone]> select * from policy_association;
Empty set (0.00 sec)

MariaDB [keystone]> select * from project;
Empty set (0.00 sec)

MariaDB [keystone]> select * from project_endpoint;
Empty set (0.00 sec)

MariaDB [keystone]> select * from project_endpoint_group;
Empty set (0.00 sec)

MariaDB [keystone]> select * from region;
+-----------+-------------+------------------+-------+
| id        | description | parent_region_id | extra |
+-----------+-------------+------------------+-------+
| RegionOne |             | NULL             | {}    |
+-----------+-------------+------------------+-------+
1 row in set (0.00 sec)

MariaDB [keystone]> select * from request_token;
Empty set (0.00 sec)

MariaDB [keystone]> select * from revocation_event;
Empty set (0.00 sec)

MariaDB [keystone]> select * from role;
Empty set (0.00 sec)

MariaDB [keystone]> select * from sensitive_config;
Empty set (0.00 sec)

MariaDB [keystone]> select * from service;
+----------------------------------+----------+---------+-----------------------------------------------------------+
| id                               | type     | enabled | extra                                                     |
+----------------------------------+----------+---------+-----------------------------------------------------------+
| 20ac0030158744a499e0c9b04ba077a5 | identity |       1 | {"description": "OpenStack Identity", "name": "keystone"} |
+----------------------------------+----------+---------+-----------------------------------------------------------+
1 row in set (0.00 sec)

MariaDB [keystone]> select * from service_provider;
Empty set (0.00 sec)

MariaDB [keystone]> select * from token;
Empty set (0.00 sec)

MariaDB [keystone]> select * from trust;
Empty set (0.00 sec)

MariaDB [keystone]> select * from trust_role;
Empty set (0.00 sec)

MariaDB [keystone]> select * from user;
Empty set (0.00 sec)

MariaDB [keystone]> select * from user_group_membership;
Empty set (0.00 sec)

MariaDB [keystone]> select * from whitelisted_config;
Empty set (0.00 sec)

MariaDB [keystone]>exit
```








<!--
自分用 cmd
mysql -u keystone -h controller01 -p

show databases;

use keystone;

show tables;

select * from access_token;
select * from assignment;
select * from config_register;
select * from consumer;
select * from credential;
select * from domain;
select * from endpoint;
select * from endpoint_group;
select * from federation_protocol;
select * from group;
select * from id_mapping;
select * from identity_provider;
select * from idp_remote_ids;
select * from mapping;
select * from migrate_version;
select * from policy;
select * from policy_association;
select * from project;
select * from project_endpoint;
select * from project_endpoint_group;
select * from region;
select * from request_token;
select * from revocation_event;
select * from role;
select * from sensitive_config;
select * from service;
select * from service_provider;
select * from token;
select * from trust;
select * from trust_role;
select * from user;
select * from user_group_membership;
select * from whitelisted_config;
-->
