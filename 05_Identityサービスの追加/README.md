# Identityサービスの追加

http://docs.openstack.org/liberty/ja/install-guide-rdo/keystone.html

OpenStack Identity サービス(Keystone) は、認証、認可、サービスカタログサービスを管理する OpenStack のサービス。他の OpenStack サービスは共通の統一 API として Identity サービスを使用出来る。

この手順では、コントローラーノードに OpenStack Identity サービス (コード名 keystone) をインストールして設定する。今回は、パフォーマンスを重視し、Apache HTTP サーバーを使ってリクエストを処理し、SQL データベースの代わりに Memcached を使ってトークンを保存する。

## データベース と 管理トークン 作成

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
# <ここでは、トークンの代わりにPassword123$とする>
# openssl rand -hex 10
========>
Password123$
========<
```


## コンポーネントのインストール と 設定

(注釈) Kilo リリースと Liberty リリースでは、keystone プロジェクトは eventlet を非推奨扱いとしています。代わりに WSGI 拡張に対応した専用 Web サーバーの使用を推奨しています。このガイドでは、Apache HTTP server の mod_wsgi を使用して、5000 番ポートと 35357 番ポートで Identity サービスのリクエストを処理します。デフォルトでは、keystone サービスは、まだ 5000 番と 35357 番をリッスンしています。そのため、このガイドでは、keystone サービスを無効化します。keystone プロジェクトは、Mitaka リリースで eventlet のサポートを削除する予定です。

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
### <No handlers could be found for logger "oslo_config.cfg" ってエラーが出る>
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


MariaDB [(none)]> show databases;
========>
+--------------------+
| Database           |
+--------------------+
| information_schema |
| keystone           |
+--------------------+
2 rows in set (0.00 sec)
========<


MariaDB [(none)]> use keystone;


MariaDB [keystone]> show tables;
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
