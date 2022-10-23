---
title: Linux添加服务到开机自动启动
date: 2018-04-22 10:20:20
tags: Linux
categories: 操作系统
---

>Systemd 是 Linux 系统中最新的初始化系统，Systemd 服务文件以 .service 结尾。一些使用包管理工具安装的软件会自动建立 .service 服务文件，路径在 /lib/systemd/system/ 下，但自行建立及管理的文件建议放在 /etc/systemd/system/ 目录下。内容以 supervisor 为例：

```
[Unit]
Description=Supervisor process control system for UNIX
Documentation=http://supervisord.org
After=network.target

[Service]
ExecStart=/usr/bin/supervisord -n -c /etc/supervisor/supervisord.conf
ExecStop=/usr/bin/supervisorctl $OPTIONS shutdown
ExecReload=/usr/bin/supervisorctl -c /etc/supervisor/supervisord.conf $OPTIONS reload
KillMode=process
Restart=on-failure
RestartSec=50s

[Install]
WantedBy=multi-user.target
```
参数说明：
```
[Unit]
Description：描述服务
Documentation：参考资料
After：描述服务类别

[Service]
Type：是后台运行的形式
ExecStart：服务的具体运行命令
ExecReload：重启命令
ExecStop：停止命令
KillMode：daemon终止时所关闭的程序
Restart：触发重启
RestartSec：重启等待时间
TimeoutSec：无法顺利启动强制关闭时间
注意：[Service]的启动、重启、停止命令全部要求使用绝对路径

[Install]
运行级别下服务安装的相关设置，可设置为多用户，即系统运行级别为3
```
如过想要某些服务开机启动，例如以 php-fpm 为例，编写service：
```
sudo vi /etc/systemd/system/php7.1-fpm.service
```
```
[Unit]
Description=The PHP 7.1 FastCGI Process Manager
Documentation=man:php-fpm7.1(8)
After=network.target

[Service]
Type=notify
PIDFile=/run/php/php7.1-fpm.pid
ExecStart=/usr/sbin/php-fpm7.1 -R --nodaemonize --fpm-config /etc/php/7.1/fpm/php-fpm.conf
ExecReload=/bin/kill -USR2 $MAINPID

[Install]
WantedBy=multi-user.target
```
随后如果 php-fpm 在运行则先将其关闭，运行：
```
systemctl daemon-reload 
systemctl start php7.1-fpm.service
```
测试能够成功开启服务了，就可以将服务设置为开机启动：
```
systemctl enable php7.1-fpm.service
```

### 常用命令
```
systemctl daemon-reload
systemctl list-units --type=service
systemctl list-unit-files --type=service

systemctl start unit.service
systemctl stop unit.service
systemctl restart unit.service

systemctl enable unit.service
systemctl disable unit.service

systemctl is-enable unit.service
systemctl is-active unit.service
systemctl status unit.service
```
