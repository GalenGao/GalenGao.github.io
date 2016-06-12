---
layout: post
title:  "mysql之mysql_config_editor"
date:   2016-05-25 14:25:20 +0700
categories: [linux, mysql]
---

mysql_config_editor允许你把登录的身份验证信息存储在一个名为.mylogin.cnf的文件里，该文件的位置在windows下是在%APPDATA%\MySQL目录下，linux下是在用户的家目录下。该文件可在以后通过MySQL客户端程序可以读取，以获得身份验证凭据用于连接到MySQL服务器。  
mysql_config_editor允许你把登录的身份验证信息存储在一个名为.mylogin.cnf的文件里，该文件的位置在windows下是在%APPDATA%\MySQL目录下，linux下是在用户的家目录下。该文件可在以后通过MySQL客户端程序可以读取，以获得身份验证凭据用于连接到MySQL服务器。  
并且，该工具至少在mysql5.6.6以上的版本才可用。  


# 创建一个login-path：    

	shell> mysql_config_editor set --login-path=test --user=root --password --host=localhost
	Enter password:

创建好后，.mylogin.cnf将保存在用户的家目录下，此处我用的是RHEL6，即/home/op下。该文件是不可读的，它类似于选项组，包含单个身份的验证信息。 

# 在登录mysql时，可以指定创建的login-path名，然后直接进入：  

    shell> mysql --login-path=test-login
    Welcome to the MySQL monitor. Commands end with ; or \g.
    Your MySQL connection id is 4
    Server version: 5.6.26-log Source distribution
    …………………………………………………………

但是如果有人能够拿到该文件，通过一些方式，是可以将其破解并获取你的密码。  

login-path只能被创建用户使用（OS层面）。  

# 如果想看.mylogin.cnf里写了什么，可以使用：  

    shell> mysql_config_editor print --all
    [test_login]
    user = root
    password = *****
    [test]
    user = root
    password = *****
    host = localhost

*当然想只看某一个则可写作* 

    shell> mysql_config_editor print --login-path=test
    [test]
    user = root
    password = *****
    host = localhost
    

# 若要删除.mylogin.cnf，则可以使用  

	shell> mysql_config_editor remove --login-path=test



其他选项：

|| *Format* || *Description* || *Introduced* ||
|| --all || Print all login paths ||	 
|| --debug[=debug_options] || Write a debugging log ||	 
|| --help || Display help message and exit ||	 
|| --host=host_name || Host to write to login file ||	 
|| --login-path=name || Login path name ||	 
|| --password || Solicit password to write to login file ||	 
|| --port=port_num || TCP/IP port number to write to login file	5.6.11 ||
|| --socket=path || The Unix socket file name to write to login file	5.6.11 ||
|| --user=user_name || User name to write to login file ||
|| --verbose || Verbose mode ||	 
|| --version || Display version information and exit ||	 
|| --warn || Warn and solicit confirmation for overwriting login path ||	 
