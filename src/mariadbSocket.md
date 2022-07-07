# MariaDB Error Connecting to Socket

> The exact error

```sh
Can't connect to a local MySQL server through socket '/var/lib/mysql/mysql.sock'
```

When I got this I read and got some good advice and I'm sure for a lot of people that probably worked, here are some of that advice:

## Completely remove MariaDB CentOS 8

I thought that no matter what the problem is with MariaDB, that completely removing it would solve the issue:

```sh
yum remove mariadb mariadb-server
rm -rf /var/lib/mysql
rm /etc/my.cnf
rm ~/.my.cnf
rm -f /var/log/mariadb
rm -f /var/log/mariadb/mariadb.log.rpmsave
rm -rf /usr/lib64/mysql
rm -rf /usr/share/mysql
```

Then reinstalling

```sh
yum install mariadb mariadb-server
mysql_secure_installation
```

But as soon as I ran `mysql_secure_installation` I got the same error.

## Then I tried resetting the root user

```sh
mariadb_safe --skip-grant-tables

# OUPUT

201014 07:35:31 mariadb_safe Logging to '/var/log/mariadb/mariadb.log'.
201014 07:35:31 mariadb_safe Starting mariadb daemon with databases from /var/lib/mysql
mysql
Welcome to the MariaDB monitor. Commands end with ; or \g.
Your MariaDB connection id is 1
Server version: 5.5.56-MariaDB MariaDB Server

Copyright (c) 2000, 2017, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
```

- Then it showed me the `MariaDB [(none)]>` prompt

Run the following cammnands

```sh
MariaDB [(none)]> UPDATE mysql.user SET Password=PASSWORD('Test@123$') WHERE User='root';
Query OK, 3 rows affected (0.00 sec)
Rows matched: 3 Changed: 3 Warnings: 0


# OUTPUT

MariaDB [(none)]> flush privileges;
Query OK, 0 rows affected (0.00 sec)
```

Then exit

```sh
MariaDB [(none)]> exit;
Bye
```

Then I ran the following command:

```sh
mysqladmin -u root -p shutdown

# Supposed to get this output
Enter password:
[1]+ Done mariadb_safe --skip-grant-tables
```

Then I'm supposed to be able to do this:

```sh
[root@localhost ~]# service mariadb start
Redirecting to /bin/systemctl start mariadb.service
```

> But again I got the same error

```sh
Can't connect to a local MySQL server through socket '/var/lib/mysql/mysql.sock'
```

## Then I tried doing it this way

```sh
#!/bin/bash
sudo systemctl stop mariadb
sudo systemctl set-environment mariadb_OPTS="--skip-grant-tables"
sudo systemctl start mariadb
sudo mysql -u root -e "UPDATE mysql.user SET authentication_string = '', password_expired = 'N' WHERE User = 'root' AND (Host = 'localhost' OR Host = '%');"
sudo mysql -u root -e "FLUSH PRIVILEGES;"
sudo mysql -u root -e "GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '<password>';"
sudo systemctl stop mariadb
sudo systemctl unset-environment mariadb_OPTS
sudo systemctl start mariadb
sudo mysql -u root -e "ALTER USER 'root'@'localhost' IDENTIFIED BY '<password>';"
sudo systemctl restart mariadb
```

> But still got the same error

## I found the problem

I ran this command and I found the issue

```sh
ls -la /var/lib/mysql/mysql.sock

# will show you the permissions and owner of the socket
```

Thanks to this post: [http://www.dark-hamster.com/database/how-to-solve-error-message-database-mariadb-is-probably-initialized-in-var-lib-mysql-already-nothing-is-done/](http://www.dark-hamster.com/database/how-to-solve-error-message-database-mariadb-is-probably-initialized-in-var-lib-mysql-already-nothing-is-done/) I realized that my permissions got all screwed up in the mysql dirs, and it was not persisting

```sh
[root@10 ~]# cd /var/lib/mysql/
[root@10 mysql]# ls
 aria_log.00000001   binlog.index      client-key.pem       ib_buffer_pool  '#innodb_temp'        performance_schema   server-key.pem
 aria_log_control    ca-key.pem        db_guest_machine     ibdata1          mysql                private_key.pem      sys
 auto.cnf            ca.pem           '#ib_16384_0.dblwr'   ib_logfile0      mysql.ibd            public_key.pem       undo_001
 binlog.000001       client-cert.pem  '#ib_16384_1.dblwr'   ib_logfile1      mysql_upgrade_info   server-cert.pem      undo_002
[root@10 mysql]# mkdir temp
```

Move all of the content inside of `/var/lib/mysql` to a certain variable. For an example `temp` as in the creation of it above. In this context, to clear all of the content, just move `/var/lib/mysql/temp` to another path. For an example, it is a certain folder and path, such as `/root`.

```sh
[root@10 mysql]# mv * temp/
mv: cannot move 'temp' to a subdirectory of itself, 'temp/temp'
[root@10 mysql]# ls -al
total 8
drwxr-xr-x.  3 mysql mysql   18 Mar 19 21:51 .
drwxr-xr-x. 39 root  root  4096 Mar 18 23:47 ..
drwxr-xr-x   7 root  root  4096 Mar 19 21:51 temp
[root@10 mysql]# mv temp/ /root/temp/mysql-temp
[root@10 mysql]# cd ..
[root@10 lib]# chown -Rv mysql.mysql /var/lib/mysql/
ownership of '/var/lib/mysql/' retained as mysql:mysql
[root@10 lib]#
```

 Next, start the MariaDB service and soon after check ‘/var/lib/mysql’ folder’s content as in the following command execution :

 ```sh
[root@10 lib]# systemctl start mariadb
[root@10 lib]# cd /var/lib/mysql/
[root@10 mysql]# ls
aria_log.00000001  ib_buffer_pool  ib_logfile0  ibtmp1             mysql       mysql_upgrade_info  tc.log
aria_log_control   ibdata1         ib_logfile1  multi-master.info  mysql.sock  performance_schema
[root@10 mysql]#
 ```

 As in the above output, starting MariaDB service is a success. Moreover, there are generated content after starting MariaDB service in `/var/lib/sql`.

 ## Conclusion

 This command saved the day:

 ```sh
 chown -Rv mysql.mysql /var/lib/mysql/
 ```