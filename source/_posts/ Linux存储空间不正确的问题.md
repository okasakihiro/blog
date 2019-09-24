---
title: Linux存储空间不正确的问题
date: 2019/9/24 16:57
categories: Linux
tag: [Linux,操作系统]
toc: true
---

最近自己的服务器经常提示磁盘空间不足，起初以为是log文件太多了，没有定期清理，但在删除了一些log文件后，发现磁盘占用率还是很高，就有了下面这篇文章：

# 查看空间占用
在Linux中，有两个查看磁盘占用空间的指令：
1. df
   * df用于显示目前在Linux系统上的文件系统的磁盘使用情况统计
   * ```shell
      work@server:~$ df
      Filesystem     1K-blocks     Used Available Use% Mounted on
      udev             2004896        0   2004896   0% /dev
      tmpfs             404644     3384    401260   1% /run
      /dev/vda1       51473020 48880496         0 100% /
      tmpfs            2023200        0   2023200   0% /dev/shm
      tmpfs               5120        4      5116   1% /run/lock
      tmpfs            2023200        0   2023200   0% /sys/fs/cgroup
      tmpfs             404644        0    404644   0% /run/user/1000
     ```
2. du
   * du用于显示目录或文件的大小。du会显示指定的目录或文件锁占用的磁盘空间
   * ```shell
      work@server:~/zentaopms$ du -sh *
      28K	bin
      44K	config
      472K	db
      276K	doc
      184K	framework
      4.0M	lib
      9.4M	module
      1.3G	tmp
      4.0K	VERSION
      176M	www
     ```

使用上面的两个命令，便可以查询磁盘占用空间的情况。

# 问题浮现
在删除前，可以看到上面的占用率已经到达了100%，说明磁盘已满，需要删除一些文件来释放空间。
在删除了一些日志文件之后，再次使用df命令：
```shell
work@server:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            2.0G     0  2.0G   0% /dev
tmpfs           396M  3.3M  392M   1% /run
/dev/vda1        50G   38G  9.0G  81% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M  4.0K  5.0M   1% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
tmpfs           396M     0  396M   0% /run/user/1000
```

可以看到已经释放了9.0G的空间，但此时在根目录中输入 `du -sh` 会显示:
```
work@server:/$ sudo du -sh
19G	.
```
来了来了，奇怪的问题环节，为什么我整个根目录只占用19G，但是 `df` 指令会显示占用了38G呢？

# 搜寻原因
一般情况下只要是删除了文件，肯定会释放空间的，但是有一部分的文件会出现例外，那就是当文件被进程锁定或存在某个进程一直在访问这个文件的时候，就会出现删除文件但没有释放空间的情况。
> 在Linux或者Unix系统中，通过rm或者文件管理器删除文件将会从文件系统的目录结构上解除链接(unlink).然而如果文件是被打开的（有一个进程正在使用），那么进程将仍然可以读取该文件，磁盘空间也一直被占用。

这便是上面出现存储空间不一致的原因。

# 如何解决

既然知道了问题的所在，那接下来我们借助 `lsof` 指令来查找这个问题的具体情况。

> lsof（list open files）是一个列出当前系统打开文件的工具。在linux环境下，任何事物都以文件的形式存在，通过文件不仅仅可以访问常规数据，还可以访问网络连接和硬件。

执行以下指令：
```shell
work@server:~$ sudo lsof | grep deleted
systemd-j   200             root  txt       REG              253,1      326224    1577888 /lib/systemd/systemd-journald (deleted)
dhclient    508             root  txt       REG              253,1      487248     393285 /sbin/dhclient (deleted)
systemd-l   564             root  txt       REG              253,1      618520    1573439 /lib/systemd/systemd-logind (deleted)
rsyslogd    567           syslog    5w      REG              253,1 18965122883    1836497 /var/log/syslog (deleted)
rsyslogd    567           syslog    7w      REG              253,1  1163682216    1836932 /var/log/auth.log (deleted)
in:imuxso   567   671     syslog    5w      REG              253,1 18965122883    1836497 /var/log/syslog (deleted)
in:imuxso   567   671     syslog    7w      REG              253,1  1163682216    1836932 /var/log/auth.log (deleted)
in:imklog   567   672     syslog    5w      REG              253,1 18965122883    1836497 /var/log/syslog (deleted)
in:imklog   567   672     syslog    7w      REG              253,1  1163682216    1836932 /var/log/auth.log (deleted)
rs:main     567   673     syslog    5w      REG              253,1 18965122883    1836497 /var/log/syslog (deleted)
rs:main     567   673     syslog    7w      REG              253,1  1163682216    1836932 /var/log/auth.log (deleted)
dbus-daem   606       messagebus  txt       REG              253,1      224208    2228589 /usr/bin/dbus-daemon (deleted)
php-fpm7.   729         www-data    4u      REG              253,1           0    2112406 /tmp/.ZendSem.ujOWMj (deleted)
systemd    1070             work  txt       REG              253,1     1577232    1577736 /lib/systemd/systemd (deleted)
(sd-pam    1074             work  txt       REG              253,1     1577232    1577736 /lib/systemd/systemd (deleted)
bash       1077             work  txt       REG              253,1     1037528     528574 /bin/bash (deleted)
php-fpm7.  5660         www-data    4u      REG              253,1           0    2112406 /tmp/.ZendSem.ujOWMj (deleted)
php-fpm7.  8952             root    4u      REG              253,1           0    2112406 /tmp/.ZendSem.ujOWMj (deleted)
python    11023             work  txt       REG              253,1     3542008    2237861 /usr/bin/python2.7 (deleted)
superviso 22597             root  txt       REG              253,1     3542008    2237861 /usr/bin/python2.7 (deleted)
redis-ser 22611            redis  txt       REG              253,1      820624    2245182 /usr/bin/redis-server (deleted)
redis-ser 22611 22617      redis  txt       REG              253,1      820624    2245182 /usr/bin/redis-server (deleted)
redis-ser 22611 22618      redis  txt       REG              253,1      820624    2245182 /usr/bin/redis-server (deleted)
php-fpm7. 32749         www-data    4u      REG              253,1           0    2112406 /tmp/.ZendSem.ujOWMj (deleted)

```
这里使用 `lsof | grep deleted` 会筛选出带有deleted标识的文件（意味着文件已经被删除，但在某些进程中依旧在使用）


根据上面可以看到 `rsyslogd` 这个进程占用了很多空间（ `log/syslog` 文件 `log/auth.log` 文件 ）


由于 `rsyslogd` 是系统进程，所以我们使用下面的指令来重启以下 `rsyslogd` ：
```shell
work@server:/$ sudo systemctl restart rsyslog
```

重启成功后，再次输入 `lsof | grep deleted` 查看文件打开情况：
```shell
work@xiangpi:~/m_pencilnews$ sudo lsof | grep deleted
systemd-j   200             root  txt       REG              253,1    326224    1577888 /lib/systemd/systemd-journald (deleted)
dhclient    508             root  txt       REG              253,1    487248     393285 /sbin/dhclient (deleted)
systemd-l   564             root  txt       REG              253,1    618520    1573439 /lib/systemd/systemd-logind (deleted)
dbus-daem   606       messagebus  txt       REG              253,1    224208    2228589 /usr/bin/dbus-daemon (deleted)
php-fpm7.   729         www-data    4u      REG              253,1         0    2112406 /tmp/.ZendSem.ujOWMj (deleted)
systemd    1070             work  txt       REG              253,1   1577232    1577736 /lib/systemd/systemd (deleted)
(sd-pam    1074             work  txt       REG              253,1   1577232    1577736 /lib/systemd/systemd (deleted)
bash       1077             work  txt       REG              253,1   1037528     528574 /bin/bash (deleted)
php-fpm7.  5660         www-data    4u      REG              253,1         0    2112406 /tmp/.ZendSem.ujOWMj (deleted)
php-fpm7.  8952             root    4u      REG              253,1         0    2112406 /tmp/.ZendSem.ujOWMj (deleted)
python    11023             work  txt       REG              253,1   3542008    2237861 /usr/bin/python2.7 (deleted)
superviso 22597             root  txt       REG              253,1   3542008    2237861 /usr/bin/python2.7 (deleted)
redis-ser 22611            redis  txt       REG              253,1    820624    2245182 /usr/bin/redis-server (deleted)
redis-ser 22611 22617      redis  txt       REG              253,1    820624    2245182 /usr/bin/redis-server (deleted)
redis-ser 22611 22618      redis  txt       REG              253,1    820624    2245182 /usr/bin/redis-server (deleted)
php-fpm7. 32749         www-data    4u      REG              253,1         0    2112406 /tmp/.ZendSem.ujOWMj (deleted)
```

可以看到，被占用的空间已经被释放了。

使用 `du` 指令查看以下磁盘使用情况：
```shell
work@server:/$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            2.0G     0  2.0G   0% /dev
tmpfs           396M  3.3M  392M   1% /run
/dev/vda1        50G   19G   28G  41% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M  4.0K  5.0M   1% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
tmpfs           396M     0  396M   0% /run/user/1000
```

至此，释放磁盘空间完成。

# 小结
说实话，之前也曾经多次遇到这个问题，每次只能删一些日志文件来临时解决，查到了磁盘空间不一致的情况，也没有针对性的去追寻其真正的原因，做一次总结，也算是学习了新的知识。