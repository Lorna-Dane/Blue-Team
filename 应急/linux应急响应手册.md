> 当企业发生黑客入侵、系统崩溃或其它影响业务正常运行的安全事件时，急需第一时间进行处理，使企业的网络信息系统在最短时间内恢复正常工作，进一步查找入侵来源，还原入侵事故过程，同时给出解决方案与防范措施，为企业挽回或减少经济损失。

> 针对常见的攻击事件，结合工作中应急响应事件分析和解决的方法，总结了一些 Linux 服务器入侵排查的思路。

[文章来源](https://blog.csdn.net/jihaichen/article/details/86756008)

# 0x01 入侵排查思路

## 一、账号安全

### 基本使用：

**1、用户信息文件 /etc/passwd**

```bash
root:x\:0:0:root:/root:/bin/bash
account:password:UID:GID:GECOS:directory:shell
```

用户名：密码：用户ID：组ID：用户说明：家目录：登陆之后shell

注意：无密码只允许本机登陆，远程不允许登陆

**2、影子文件 /etc/shadow**

```bash
root:$6$oGs1PqhL2p3ZetrE$X7o7bzoouHQVSEmSgsYN5UD4.kMHx6qgbTqwNVC5oOAouXvcjQSt.Ft7ql1WpkopY0UV9ajBwUt1DpYxTCVvI/:16809:0:99999:7::: 
//用户名：加密密码：密码最后一次修改日期：两次密码的修改时间间隔：密码有效期：密码修改到期到的警告天数：密码过期之后的宽限天数：账号失效时间：保留
```

**3、几个常用命令：**

```bash
who      //查看当前登录用户（tty 本地登陆 pts 远程登录）
w        //查看系统信息，想知道某一时刻用户的行为
uptime   //查看登陆多久、多少用户，负载
```

### 入侵排查：

**1、查询特权用户(特权用户uid 为 0)**

查看`/etc/passwd`中UID为0的账户。


**2、查询可以远程登录的帐号信息**

无密码不能远程登陆。打开`/etc/shadow`中查看有加密密码的用户进行判断。(配合`who`、`uptime`进行判断哪些正在被远程)

**3、禁用或删除多余及可疑的帐号**

```bash
usermod -L user   //禁用帐号，帐号无法登录，/etc/shadow 第二栏为 ! 开头
userdel user      //删除 user 用户
userdel -r user   //将删除user用户，并且将/home目录下的user目录一并删除
```

## 二、历史命令

### 基本使用：

> 通过 .bash_history 查看帐号执行过的系统命令

**1、root 的历史命令**

```bash
history
```

**2、历史操作命令的清除：**

```bash
history -c
```

ps:但此命令并不会清除保存在文件中的记录，因此需要手动删除 `/用户名/.bash_profile` 文件中的记录。

### 入侵排查：

进入用户目录下:

```bash
cat .bash_history >> history.txt  //在用户根目录下导出history.txt
```

## 三、端口

使用 netstat 网络连接命令，分析可疑端口、IP、PID

```bash
netstat -antlp | more
```

ps:在linux上没有netstat命令时，使用`yum install net-tools`安装。

运行 ls -l /proc/$PID/exe 或 file /proc/$PID/exe（$PID 为对应的 pid 号）

## 四、进程

使用 ps 命令，分析进程

```bash
ps aux | grep pid     //输入想查看的pid数字
```

ps：也可以使用top查看高使用率的进程进行分析。

## 五、开机启动项

### 基本使用：

系统运行级别示意图：

![](../img/linux应急img2.png)

查看运行级别命令

```bash
runlevel
```

系统默认允许级别

```bash
vi /etc/inittab id=3：initdefault   //系统开机后直接进入哪个运行级别
```

开机启动配置文件

```bash
/etc/rc.local 
/etc/rc.d/rc[0~6].d
```

例子:当我们需要开机启动自己的脚本时，只需要将可执行脚本丢在 /etc/init.d 目录下，然后在 /etc/rc.d/rc*.d 中建立软链接即可

```bash
[root@localhost ~]# ln -s /etc/init.d/sshd /etc/rc.d/rc3.d/S100ssh
```

此处 `sshd` 是具体服务的脚本文件，`S100ssh` 是其软链接，`S` 开头代表加载时自启动；如果是 `K` 开头的脚本文件，代表运行级别加载时需要关闭的。

### 入侵排查：

启动项文件：

```bash
more /etc/rc.local /etc/rc.d/rc[0~6].d ls -l /etc/rc.d/rc3.d/
```

查看是否存在脚本的软链接。

## 六、定时任务

### 基本使用：

**1、利用 crontab 创建计划任务**

crontab -l 列出某个用户 cron 服务的详细内容

> Tips：默认编写的 crontab 文件会保存在 ( /var/spool/cron/用户名 例如: /var/spool/cron/root）

crontab -r 删除每个用户 cront 任务`(谨慎：删除所有的计划任务)`

crontab -e 使用编辑器编辑当前的 crontab 文件

如：*/1 * * * * echo "hello world" >> /tmp/test.txt 每分钟写入文件

格式如下

```
*    *    *    *    *
-    -    -    -    -
|    |    |    |    |
|    |    |    |    +----- 星期中星期几 (0 - 7) (星期天 为0)
|    |    |    +---------- 月份 (1 - 12) 
|    |    +--------------- 一个月中的第几天 (1 - 31)
|    +-------------------- 小时 (0 - 23)
+------------------------- 分钟 (0 - 59)
```

**2、利用 anacron 实现异步定时任务调度**

每天运行 /home/backup.sh 脚本：

```bash
vi /etc/anacrontab @daily 10 example.daily /bin/bash /home/backup.sh
```

当机器在 backup.sh 期望被运行时是关机的，anacron 会在机器开机十分钟之后运行它，而不用再等待 7 天。

解释：
anacron 如何在 Linux 工作？

anacron 任务被列在 `/etc/anacrontab` 中，任务可以使用下面的格式(anacron 文件中的注释必须以 `#` 号开始)安排。

anacron 用于以天为单位的频率运行命令。它的工作与 cron 稍有不同，它假设机器不会一直开机。

anacron 会检查任务是否已经在 period(某一时间) 字段指定的时间被执行了。如果没有，则在等待 delay(延迟) 字段中指定的分钟数后，执行 command 字段中指定的命令。 一旦任务被执行了，它会使用 job-id(时间戳文件名)字段中指定的名称将日期记录在 /var/spool/anacron 目录中的时间戳文件中。

### 入侵排查：

重点关注以下目录中是否存在恶意脚本

```bash
/var/spool/cron/* 
/etc/crontab 
/etc/cron.d/* 
/etc/cron.daily/* 
/etc/cron.hourly/* 
/etc/cron.monthly/* 
/etc/cron.weekly/ 
/etc/anacrontab 
/var/spool/anacron/*
```

*小技巧：*

```bash
more + 路径目录    //查看目录下所有文件
```

## 七、服务

### 服务自启动：

**第一种修改方法：**

```bash
chkconfig [--level 运行级别][独立服务名][on|off]
chkconfig –level 2345 httpd on 开启自启动
chkconfig httpd on （默认 level 是 2345）
```

**第二种修改方法：**

```bash
修改 /etc/re.d/rc.local 文件 加入 /etc/init.d/httpd start
```

**第三种修改方法：**

使用 ntsysv 命令管理自启动，可以管理独立服务和 xinetd 服务。

ps:如果没有ntsysv命令使用`yum install ntsysv`安装。

### 入侵排查：

**1、查询已安装的服务：**

*RPM 包安装的服务:*

```bash
chkconfig --list 查看服务自启动状态，可以看到所有的RPM包安装的服务

ps aux | grep "服务名字" 查看当前服务（不加引号）
```

*系统在 3 与 5 级别下的启动项*

中文环境

```bash
chkconfig --list | grep "3:启用|5:启用"
```

英文环境

```bash
chkconfig --list | grep "3:on|5:on"
```

*源码包安装的服务*

查看服务安装位置 ，一般是在 /user/local/

```bash
service httpd start
```

搜索 /etc/rc.d/init.d/ 查看是否存在

## 八、系统日志

日志默认存放位置：/var/log/

查看日志配置情况：more /etc/rsyslog.conf

![](../img/linux应急img3.png)

**主要日志文件介绍：**

```bash
内核及公共消息日志:/var/log/messages
计划任务日志：/var/log/cron
系统引导日志：/var/log/dmesg
邮件系统日志:/var/log/maillog
用户登录日志：/var/log/lastlog
/var/log/boot.log（记录系统在引导过程中发生的时间）
/var/log/secure (用户验证相关的安全性事件)
/var/log/wtmp(当前登录用户详细信息)
/var/log/btmp（记录失败的的记录）
/var/run/utmp（用户登录、注销及系统开、关等事件）
```

**日志文件详细介绍：**

1、
```bash
/var/log/secure        
//Linux系统安全日志，记录用户和工作组的情况、用户登陆认证情况
```

![](../img/linux应急img4.png)

可以看到这里有ssh的连接日志。

2、
```bash
/var/log/boot.log      
//该文件记录了系统在引导过程中发生的事件
```

![](../img/linux应急img5.png)

说实话，这个用处并不是很大。

3、
```bash
/var/log/messages
//内核及公共信息日志，是许多进程日志文件的汇总，从该文件中可以看出任何***或成功的***
```

![](../img/linux应急img6.png)

里面的内容很多，而且很杂，一般情况下我们用不到这个。

4、
```bash
/var/log/dmesg
//系统引导日志
```

该日志使用`dmesg`命令快速查看最后一次系统引导的引导日志
dmesg | more

![](../img/linux应急img7.png)

5、
```bash
/var/log/lastlog
//最近的用户登录事件，一般记录最后一次的登录事件
```

ps:该日志不能用诸如cat、tail等查看，因为该日志里面是二进制文件，可以用`lastlog`命令查看，它根据UID排序显示登录名、端口号（tty）和上次登录时间。如果一个用户从未登录过，lastlog显示 `Never logged`。

![](../img/linux应急img8.png)

6、
```bash
/var/log/wtmp
//该日志文件永久记录每个用户登录、注销及系统的启动、停机的事件。
```

ps:该日志为二进制文件，不能用诸如tail/cat/等命令，使用last命令查看。

![](../img/linux应急img9.png)

7、
```bash
/var/log/mailog
//记录邮件的收发
```

![](../img/linux应急img10.png)

8、
```bash
/var/log/btmp
//此文件是记录错误登录的日志，可以记录有人使用暴力破解ssh服务的日志。
```

ps:该文件用lastb打开

![](../img/linux应急img11.png)

9、
```bash
/var/log/utmp
//该日志记录当前用户登录的情况，不会永久保存记录。
```

ps:可以用who/w命令来查看

![](../img/linux应急img12.png)
