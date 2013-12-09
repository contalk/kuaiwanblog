Title: splunk安装介绍
Date: 2013-12-05 15:30
Category: 数据分析
Tags: linux,splunk
slug: splunk-install
Author: chenguo

### 1. 安装

1) 下载rpm包：
[splunk企业版](http://download.splunk.com/releases/6.0/splunk/linux/splunk-6.0-182037-linux-2.6-x86_64.rpm)
[splunk转发代理](http://download.splunk.com/releases/6.0/universalforwarder/linux/splunkforwarder-6.0-182037-linux-2.6-x86_64.rpm)

2) 安装：
安装splunk企业版:

    rpm -i --prefix=/opt splunk-6.0-182037-linux-2.6-x86_64.rpm
    export SPLUNK_HOME=/opt/splunk
    export PATH=$SPLUNK_HOME/bin:$PATH

安装splunk转发代理:

    rpm -i --prefix=/opt splunkforwarder-6.0-182037-linux-2.6-x86_64.rpm
    export SPLUNK_HOME=/opt/splunk
    export PATH=$SPLUNK_HOME/bin:$PATH

### 2. 配置

1.配置数据转发:
配置转发代理:

    cd $SPLUNK_HOME/bin
    #设置开机启动
    ./splunk enable boot-start
    #配置转发目的地址及端口
    ./splunk add forward-server 10.0.0.175:9999
    #配置监控目录或文件
    ./splunk add monitor $DIR
    #开启转发
    ./splunk start

2.配置接收方:
进入web管理界面进行配置 
[官网教程](http://docs.splunk.com/Documentation/Splunk/latest/Forwarding/Setupforwardingandreceiving)

![配置接收](/images/chenguo/setting_forward.png)

进入后点击配置接收，设定接收端口。

### 3. 登陆splunk的web管理界面：
1.进入web管理页面 http://$splunk-host:8000

* 首次登陆:
* 用户名 : admin
* 密  码 : changeme

2.登陆成功后修改密码.

在处理某类型数据时，可以将一小段数据文件导入，以供splunk学习，在你的帮助下，splunk将自动将日志解析为你想要的格式。

上方导航栏点击：

* 设置=> 数据导入
* =>添加数据
* =>从文件和目录

![添加数据](/images/chenguo/add_data.png)

然后开始调教你的splunk吧。学习完成后，让转发代理监控该类型的数据文件目录，splunk将会索引该类型的数据源，

### 4.开始搜索

[详细搜索教程官网](http://docs.splunk.com/Documentation/Splunk/latest/SearchTutorial/WelcometotheSearchTutorial)