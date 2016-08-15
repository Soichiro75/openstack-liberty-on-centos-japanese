# 04_サービス事前準備

*** 自分のリポジトリ OpenStack on Ubuntu オンプレ からコピペ。Cent用に書き直し中 ***

MariaDB、RabbitMQ の準備



## SQL データベース

自動起動とかいらないの？？？？？？？？？？？？？

http://docs.openstack.org/mitaka/ja/install-guide-ubuntu/environment-sql-database.html

ほとんどの OpenStack のサービスは、情報を保存するために SQL データベースを使用する。データベースは、一般的にコントローラーノードで実行する。 この手順では MariaDB を使用する。OpenStack のサービスは、MySQL、 PostgreSQL などの他の SQL データベースもサポートしている。

### データベースのインストール

- パッケージのインストール
[対象: controller01]

```
# apt-get install mariadb-server python-pymysql
  Do you want to continue? [Y/n] Y
  #####################
  本来ここでrootパスワードを聞かれるはず。。。。。。。。。だが、聞かれなかった。。。。。。
```


### データベース の rootパスワード設定

- rootパスワード設定
[対象: controller01]

```
# <MariaDBにログイン>
# <最初のログイン時はパスワードが設定されていない>
# mysql -u root


# <ユーザーとパスワード確認>
# SELECT User, Host, Password FROM mysql.user;
========>
+------+-----------+----------+
| User | Host      | Password |
+------+-----------+----------+
| root | localhost |          |
+------+-----------+----------+
1 row in set (0.00 sec)
========<


# <rootパスワードの設定>
MariaDB [(none)]> UPDATE mysql.user SET Password = PASSWORD('Password123$') WHERE User = 'root';


# <ユーザーとパスワード確認>
MariaDB [(none)]> SELECT User, Host, Password FROM mysql.user;
========>
+------+-----------+-------------------------------------------+
| User | Host      | Password                                  |
+------+-----------+-------------------------------------------+
| root | localhost | *6C12FC2B651683A59ADD31A5C49E1E6F801D399F |
+------+-----------+-------------------------------------------+
1 row in set (0.00 sec)
========<


# <変更の反映>
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> exit;

# <ログイン 確認>
# mysql -u root -p
Enter password: Password123$
MariaDB [(none)]> exit;

############################## 自分メモ 後で対応
######################
なぜかパスワードなしでもログイン出来てしまう。。。。
mysql -u root
MariaDB [(none)]> exit;

パスワードを間違えてもログイン出来てしまう。。。。
mysql -u root -p
Enter password: aaaaaaaaaaaaaaaa
MariaDB [(none)]> exit;

何か、おかしい。。。。。
cat /etc/mysql/debian.cnf
========>
# Automatically generated for Debian scripts. DO NOT TOUCH!
[client]
host     = localhost
user     = root
password =             # <==== 本来ここにパスワードが入っているべきなのに入っていない
socket   = /var/run/mysqld/mysqld.sock
[mysql_upgrade]
host     = localhost
user     = root
password =             # <==== 本来ここにパスワードが入っているべきなのに入っていない
socket   = /var/run/mysqld/mysqld.sock
basedir  = /usr
========<


パスワードに関する問題事象は多発しているようだ。。。。。
https://jyn.jp/ubuntu-16-04-mariadb-password-bug/


今、Ubutnu16でやっているので、
試しに、
Ubutnu14でやり直し中。。。。。。。。
```


### データベース のオプション設定

- openstack.cnf 作成
[対象: controller01]

```
# <openstack.cnf の新規作成>
# <"xxx.cnf"とは、MariaDB(MySQL)のオプション設定記述ファイル>
# <"bind-address = " のIPは "コントローラーノードの管理 IPアドレス"に置き換える。管理ネットワーク経由で他のノードにからのアクセスを可能にする>
# <"utf-8"でない場合、OpenStackモジュールとデータベース間の通信でエラーが発生してしまう。>
# vi /etc/mysql/conf.d/openstack.cnf
========> 以下を記述
[mysqld]
bind-address = 192.168.101.1
default-storage-engine = innodb
innodb_file_per_table
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
========> ここまで
```

- 設定の反映
[対象: controller01]

```
# <データベースの再起動>
# service mysql restart
```

- セキュリティの向上
[対象: controller01]

```
# < "/usr/bin/mysql_secure_installation" は以下を実施するスクリプト
#　・rootユーザーのパスワード設定
#　・アノニマスユーザーの削除
#　・rootユーザーのリモートログイン禁止
#　・テスト用データーベースの削除
#　・変更した情報の再読み込み
# >
# mysql_secure_installation
  Enter current password for root (enter for none): Password123$
  Change the root password? [Y/n] n
  Remove anonymous users? [Y/n] Y
  Disallow root login remotely? [Y/n] n
  Remove test database and access to it? [Y/n] Y
  Reload privilege tables now? [Y/n] Y
```

- MariaDBのバージョン確認
[対象: controller01]
# mysql --version
========>
mysql  Ver 15.1 Distrib 10.0.25-MariaDB, for debian-linux-gnu (x86_64) using readline 5.2
========<


### データベースのクライアント
[対象: compute01, cli01]

この手順(on 日本仮想化技術 の手順書)いらないかも？？？？？

apt-get install python-pymysql の方が良いのかも？？？？

- MariaDB クライアントのインストール

```
# <上記 "mysql --version" で調べたバージョンと同じバージョンを指定>
# apt-get install -y mariadb-client-10.0 mariadb-client-core-10.0
```


## NoSQL インストール

http://docs.openstack.org/mitaka/ja/install-guide-ubuntu/environment-nosql-database.html

Telemetry サービスは、情報を保存するために NoSQL データベースを使用する。このデータベースは一般的にコントローラーノードで実行する。OpenStackのドキュメントでは MongoDB を使用しているが、本手順では Telemetery は使用しないため、NoSQLのインストールはしない。

補足:
Telemetry サービス (コード名 ceilometer): OpenStack サービスから計測項目を収集する。必要に応じてアラームを上げる事が出来る。



## メッセージキュー インストール

http://docs.openstack.org/mitaka/ja/install-guide-ubuntu/environment-messaging.html

OpenStack は、サービス間での操作と状態をやり取りするのに、メッセージキュー を使用する。メッセージキューサービスは、一般的にコントローラーノードで動作する。OpenStack は RabbitMQ, Qpid, ZeroMQ などのメッセージキューサービスをサポートしている。しかし、ディストリビューションによりサポートされているメッセージキューサービスは異なる。この手順では、ほとんどのディストリビューションがサポートする RabbitMQ メッセージキューサービスを導入する。


- パッケージのインストールします。
[対象: controller01]

```
# apt-get install -y rabbitmq-server
```


- openstack ユーザーの追加
[対象: controller01]

```
# rabbitmqctl add_user openstack Password123$
```


- openstack ユーザーへの、設定/書き込み/読み出し の許可
[対象: controller01]

```
# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```


- 待ち受けるIPアドレスとポート の設定

ここ違う、、、、、書き直し予定。。。。。。

```
# vi /etc/rabbitmq/rabbitmq-env.conf
########> 以下を参考に編集
NODENAME=controller01
NODE_IP_ADDRESS=192.168.101.1
NODE_PORT=5672
########<
```

- RabbitMQ の再起動 と 起動確認

```
# ls -l /var/log/rabbitmq/
========>
total 8
-rw-r--r-- 1 rabbitmq rabbitmq 2160 Aug  3 17:14 rabbit@controller01.log
-rw-r--r-- 1 rabbitmq rabbitmq    0 Aug  3 17:13 rabbit@controller01-sasl.log
-rw-r--r-- 1 rabbitmq rabbitmq    0 Aug  3 17:13 startup_err
-rw-r--r-- 1 rabbitmq rabbitmq  350 Aug  3 17:13 startup_log
========<

# service rabbitmq-server stop

# service rabbitmq-server stop

# ls -l /var/log/rabbitmq/
========>
total 12
-rw-r--r-- 1 rabbitmq rabbitmq 1969 Aug  3 17:19 controller01.log
-rw-r--r-- 1 rabbitmq rabbitmq    0 Aug  3 17:19 controller01-sasl.log
-rw-r--r-- 1 rabbitmq rabbitmq 2160 Aug  3 17:14 rabbit@controller01.log
-rw-r--r-- 1 rabbitmq rabbitmq    0 Aug  3 17:13 rabbit@controller01-sasl.log
-rw-r--r-- 1 rabbitmq rabbitmq    0 Aug  3 17:19 startup_err
-rw-r--r-- 1 rabbitmq rabbitmq  336 Aug  3 17:19 startup_log
========<

# tail -f /var/log/rabbitmq/
```
