# 04_サービス事前準備

*** 自分のリポジトリ OpenStack on Ubuntu オンプレ からコピペ。Cent用に書き直し中 ***

MariaDB、RabbitMQ の準備


## SQL データベース

http://docs.openstack.org/liberty/ja/install-guide-rdo/environment-sql-database.html

ほとんどの OpenStack のサービスは、情報を保存するために SQL データベースを使用する。データベースは、一般的にコントローラーノードで実行する。 この手順では MariaDB を使用する。OpenStack のサービスは、MySQL、 PostgreSQL などの他の SQL データベースもサポートしている。

### データベースのインストール

- パッケージのインストール [対象: controller01]

```
# yum install -y mariadb mariadb-server MySQL-python
========>
(省略)
Installed:
  MySQL-python.x86_64 0:1.2.3-11.el7             mariadb.x86_64 1:5.5.50-1.el7_2             mariadb-server.x86_64 1:5.5.50-1.el7_2

Dependency Installed:
  perl.x86_64 4:5.16.3-286.el7                  perl-Carp.noarch 0:1.26-244.el7                perl-Compress-Raw-Bzip2.x86_64 0:2.061-3.el7
  perl-Compress-Raw-Zlib.x86_64 1:2.061-4.el7   perl-DBD-MySQL.x86_64 0:4.023-5.el7            perl-DBI.x86_64 0:1.627-4.el7
  perl-Data-Dumper.x86_64 0:2.145-3.el7         perl-Encode.x86_64 0:2.51-7.el7                perl-Exporter.noarch 0:5.68-3.el7
  perl-File-Path.noarch 0:2.09-2.el7            perl-File-Temp.noarch 0:0.23.01-3.el7          perl-Filter.x86_64 0:1.49-3.el7
  perl-Getopt-Long.noarch 0:2.40-2.el7          perl-HTTP-Tiny.noarch 0:0.033-3.el7            perl-IO-Compress.noarch 0:2.061-2.el7
  perl-Net-Daemon.noarch 0:0.48-5.el7           perl-PathTools.x86_64 0:3.40-5.el7             perl-PlRPC.noarch 0:0.2020-14.el7
  perl-Pod-Escapes.noarch 1:1.04-286.el7        perl-Pod-Perldoc.noarch 0:3.20-4.el7           perl-Pod-Simple.noarch 1:3.28-4.el7
  perl-Pod-Usage.noarch 0:1.63-3.el7            perl-Scalar-List-Utils.x86_64 0:1.27-248.el7   perl-Socket.x86_64 0:2.010-3.el7
  perl-Storable.x86_64 0:2.45-3.el7             perl-Text-ParseWords.noarch 0:3.29-4.el7       perl-Time-HiRes.x86_64 4:1.9725-3.el7
  perl-Time-Local.noarch 0:1.2300-2.el7         perl-constant.noarch 0:1.27-2.el7              perl-libs.x86_64 4:5.16.3-286.el7
  perl-macros.x86_64 4:5.16.3-286.el7           perl-parent.noarch 1:0.225-244.el7             perl-podlators.noarch 0:2.5.1-3.el7
  perl-threads.x86_64 0:1.87-4.el7              perl-threads-shared.x86_64 0:1.43-6.el7

Complete!
========<
```


### 確認

- 確認 [対象: controller01]

```
# <インストール確認>
# rpm -qa | grep -i "mariadb"
========>
mariadb-libs-5.5.50-1.el7_2.x86_64
mariadb-server-5.5.50-1.el7_2.x86_64
mariadb-5.5.50-1.el7_2.x86_64
========<


# <MariaDBのデフォルト設定ファイル>
# cat /etc/my.cnf
========>
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
# Settings user and group are ignored when systemd is used.
# If you need to run mysqld under a different user or group,
# customize your systemd unit file for mariadb according to the
# instructions in http://fedoraproject.org/wiki/Systemd

[mysqld_safe]
log-error=/var/log/mariadb/mariadb.log
pid-file=/var/run/mariadb/mariadb.pid

#
# include all files from the config directory
#
!includedir /etc/my.cnf.d
========<


# <上記より、/etc/my.cnf.d に設定を置く事が分かる>
# ls -l /etc/my.cnf.d
========>
total 12
-rw-r--r--. 1 root root 295 Jun 17 04:08 client.cnf
-rw-r--r--. 1 root root 232 Jun 17 04:08 mysql-clients.cnf
-rw-r--r--. 1 root root 744 Jun 17 04:08 server.cnf
========<
```

### 設定

-  設定 [対象: controller01]

```

# <"bind-address" にコントローラーノードの管理 IP アドレスを設定、管理ネットワーク経由で他のノードによりアクセスできるようにする>
# <"utf-8"にセットする。"utf-8"でない場合、OpenStackモジュールとデータベース間の通信でエラーが発生してしまう らしい>
# vi /etc/my.cnf.d/mariadb_openstack.cnf
========> 以下の内容で新規作成
[mysqld]
bind-address = 192.168.101.11
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8

========<
```

### データベースの起動と自動起動

- データベースの起動と自動起動 [対象: controller01]

```
# systemctl enable mariadb.service
========>
Created symlink from /etc/systemd/system/multi-user.target.wants/mariadb.service to /usr/lib/systemd/system/mariadb.service.
========<


# systemctl is-enabled mariadb.service
========>
enabled
========<


# systemctl start mariadb.service


# systemctl status mariadb.service
========>
● mariadb.service - MariaDB database server
   Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2016-08-15 14:14:56 JST; 25s ago
(省略)
========<
```

- MariaDBのバージョン確認 [対象: controller01]

```
# mysql --version
========>
mysql  Ver 15.1 Distrib 5.5.50-MariaDB, for Linux (x86_64) using readline 5.1
========<
```


### セキュリティ設定

- セキュリティの向上 [対象: controller01]

```
# < "/usr/bin/mysql_secure_installation" は以下を実施するスクリプト
#　・rootユーザーのパスワード設定
#　・アノニマスユーザーの削除
#　・rootユーザーのリモートログイン禁止
#　・テスト用データーベースの削除
#　・変更した情報の再読み込み
# >
# mysql_secure_installation
========>
/usr/bin/mysql_secure_installation: line 379: find_mysql_client: command not found

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none): <空ENTER>
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] Y
New password: Password123$        <=== 入力中は表示されない
Re-enter new password: Password123$        <=== 入力中は表示されない
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] Y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] n
 ... skipping.

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] Y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
========<
```


<!---
### データベースのクライアント
[対象: compute01, cli01]

この手順(on 日本仮想化技術 の手順書)いらないかも？？？？？

apt-get install python-pymysql の方が良いのかも？？？？

- MariaDB クライアントのインストール

```
# <上記 "mysql --version" で調べたバージョンと同じバージョンを指定>
# apt-get install -y mariadb-client-10.0 mariadb-client-core-10.0
```
--->


### 接続確認(local)

- 接続確認 [対象: controller01]

```
# mysql -u root -p
========>
Enter password: Password123$    <== 入力中は表示されない
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 15
Server version: 5.5.50-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
========<


# <ユーザーとパスワード確認>
MariaDB [(none)]> SELECT User, Host, Password FROM mysql.user;
========>
+------+--------------+-------------------------------------------+
| User | Host         | Password                                  |
+------+--------------+-------------------------------------------+
| root | localhost    | *6C12FC2B651683A59ADD31A5C49E1E6F801D399F |
| root | controller01 | *6C12FC2B651683A59ADD31A5C49E1E6F801D399F |
| root | 127.0.0.1    | *6C12FC2B651683A59ADD31A5C49E1E6F801D399F |
| root | ::1          | *6C12FC2B651683A59ADD31A5C49E1E6F801D399F |
+------+--------------+-------------------------------------------+
4 rows in set (0.00 sec)
========<


# <Database確認>
MariaDB [(none)]> SHOW DATABASES;
========>
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
3 rows in set (0.00 sec)
========<


# <DBから抜ける>
========>
MariaDB [(none)]> exit;
Bye
[root@controller01 ~]#
========<
```
