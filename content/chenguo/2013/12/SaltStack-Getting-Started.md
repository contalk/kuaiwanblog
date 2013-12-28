Title: SaltStack入门介绍
Date: 2013-12-19
Category: 后台开发
Tags: linux,salt,python
slug: SaltStack-Getting-Started
Author: chenguo
### 1. 简介
写在最前面：作为才了解saltstack的菜鸟，将学习配置使用saltstack的过程记录下来，留个脚印，仅供入门级别的参考。
如果想要深入理解使用salt，附上以下传送门：

[saltstack官网](http://saltstack.org/)

[salt官方手册](http://docs.saltstack.com/)

[salt states](https://github.com/saltstack/salt-states)

[中国SaltStack用户组(CSSUG)](http://saltstack.cn/)

[中国SaltStack用户组-知识库](http://wiki.saltstack.cn/)

[CSSUG仓库](https://github.com/cssug)

[Into The Salt Mine](http://intothesaltmine.org)

[Jinja2 docs](http://jinja.pocoo.org/docs/)

[YAML](http://www.yaml.org/)

下面才是开始...
	
####What is SaltStack?
Salt is a new approach to infrastructure management. Easy enough to get running in minutes, scalable enough to manage tens of thousands of servers, and fast enough to communicate with them in seconds.

Salt delivers a dynamic communication bus for infrastructures that can be used for orchestration, remote execution, configuration management and much more.

salt主要分为2部分：salt-master、salt-minion
可以这么理解master为server而minion为client


### 2. 安装

安装环境：centos6.5<br>
python版本:系统自带python2.6.6<br>
开始前确保机器可以连上互联网<br>
在这里我们将master和minion都安装在同一台机器上。

安装salt-master:

    wget -O - http://bootstrap.saltstack.org | sudo sh

安装salt-minion

    curl -L http://bootstrap.saltstack.org | sudo sh -s -- -M -N

### 3. 配置
安装完成后，我们需要对master进行配置，并启动master。
#### * 配置master服务器

    vim /etc/salt/master
    //找到以下行：#interface: 0.0.0.0
    //去掉注释修改为：
    interface: $your_ip
    /*注意，配置文件变量的冒号后面都要接一个空格，
    否则启动salt命令时，会使它解析配置文件的时候抛出异常。*/
    //然后保存。
    //重启服务
    pkill salt-master
    salt-master -d

#### * 配置被监控服务器

    vim /etc/salt/minion
    //修改以下行#master: salt
    //去掉注释修改为：
    master: $master_ip
    //给你的机器取个代号
    id: salt-master
    /*这里我将minion和master放在同一台机器上，取个这个代号表示这是master*/
    pkill salt-minion
    salt-minion -d

#### * 配置证书
现在你的minion 已经知道到master在哪里，下面开始要验证master和minion之间的通信密钥。

    //列出所有的证书，类型为：已验证/未验证/拒绝
    [root@rick salt]# salt-key -L      
    Accepted Keys:
    Unaccepted Keys:
    salt-master
    Rejected Keys:

    //验证它
    [root@rick salt]# salt-key -a salt-master
    The following keys are going to be accepted: 
    Unaccepted Keys:
    salt-master
    Proceed? [n/Y] y
    Key for minion salt-master accepted.

#### * 通讯测试
证书验证后，我们使用一下命令测试通讯是否正常：

    [root@rick salt]# salt 'salt-master' test.ping
    salt-master:
        True
输出如上返回True则恭喜，通讯测试通过。<br>
其中命令：salt 'salt-master' test.ping<br>
语法结构：salt 目标 模块.命令 [参数]

其中目标支持通配符‘*’

### 4. 使用
下面开始尝试使用salt进行配置管理...
#####待更新

