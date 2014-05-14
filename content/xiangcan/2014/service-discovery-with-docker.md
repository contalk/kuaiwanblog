title: 用docker做服务发现
slug: service-discovery-with-docker
date: 2014-04-04 12:32
tags: 后台开发, 架构
category: 后台开发
author: 火山

# docker

[在Ubuntu下安装docker](http://docs.docker.io/en/latest/installation/ubuntulinux/)

常用命令：

 - 拉取image：docker pull
 - 运行image：docker run --env -d 
 - 启动/停止contrainer：docker start[stop] -a -i
 - 进入container：docker attach 
 - 检查container：docker inspect
 - 列出container：docker ps
 - 删除container: docker rm
 - 提交image变更：docker commit old new
 - 推送image：docker push xxx/xxx

注意以下几点：
	
1. 如果不指定名字，每次docker run后会新增一个container
2. 如果指定了名字，用docker run的时候名字不能重复使用的
3. container的名字，可以用name也可以用UUID的前几位
4. container退出后，参数是会保存下来的，后续启动不需要再次输入参数
5. 当环境装好了之后，最好保存无entry point的image，否则比较难在这个基础上继续更改

docker现在尚未release 1.0，有些风险。

# skydock

[skydock项目地址](https://github.com/crosbymichael/skydock)

根据项目文档可以很快装好软件。

安装测试要注意几个问题：

1. 保证docker0的IP地址为：172.17.42.1，如果不是用ifconfig docker0 172.17.42.1 netmask 255.255.0.0
2. 如果重新设置了docker0 的IP，注意重新启动docker后container才可以ping得通
3. 运行redis-cli时，传入dns参数：--dns=172.17.42.1 ，否则域名解析不到，这里浪费掉我好多时间

实际运行注意的问题：

1. 服务的格式为：{service_name}.{environment}.kw，当然前提是运行skydns时指定-domain=kw
2. skydock是没有地方可以指定service_name的；他是使用repository的名称的。比如：fabware/grounduser，那么service_name就是grounduser

skydock通过遍历docker正在运行的container信息设置DNS，所以如果它挂掉了DNS就无法更新了。

# REST框架的更改

给urllib2打一个dns的patch就ok了，代码如下：

	:::python
	from http_patch import urllib2
	...
	
http_path.py源代码如下：

    :::python
	# -*- coding: utf-8 -*-

    import time
    import socket
    import ssl
    import random

    import urllib2
    import httplib

    import dns.name
    import dns.message
    import dns.query
    import dns.flags
    import dns.resolver

    def resolve(domain):
        resolver = dns.resolver.get_default_resolver()
        nameserver = resolver.nameservers[0]
        ADDITIONAL_RDCLASS = 65535

        domain = dns.name.from_text(domain)
        if not domain.is_absolute():
            domain = domain.concatenate(dns.name.root)

        request = dns.message.make_query(domain, dns.rdatatype.ANY)
        request.flags |= dns.flags.AD
        request.find_rrset(request.additional, dns.name.root, ADDITIONAL_RDCLASS,
                   dns.rdatatype.OPT, create=True, force_unique=True)
        response = dns.query.udp(request, nameserver)


        records = {}

        # 从anwser回复中获取数据
        for a in response.answer:
            for s in a:
                records.setdefault(s.target.to_unicode(), [])
                records[s.target.to_unicode()] = [None, s.port, None, s.priority, s.weight]

        # 从additional回复中获取数据
        for r in response.additional:
            record = records[r.name.to_unicode()]
            record[0] = r[0].to_text() # IP address
            record[2] = r.ttl

        records = records.values()

        # choose host by weight
        rand_choice = random.randint(1, 100000)
        w_total = sum([ r[4] for r in records ])
        w_lower = 0
        for r in records:
            w_higher = w_lower + 100000 * r[4] / float(w_total)
            if rand_choice >= w_lower and rand_choice < w_higher:
                return r[:3]
            w_lower = w_higher
        return random.choice(records)[:3]

    class DnsServiceResolver(object):

        def __init__(self, service_host, cache_ttl=5):
            self.service_host = service_host
            self.cache = []
            self.cache_start_time = time.time()
            self.cache_ttl = cache_ttl
            # 因为docker提供了修改dns的方法，不需要在代码中hack
            # self.resolver.nameservers.append(nameserver)

        def resolve(self):
            if self.cache and time.time() - self.cache_start_time < self.cache_ttl - 1:
                # print '>>> get from cache', self.cache, 'ttl left: ', time.time() - self.cache_start_time
                return self.cache

            record = resolve(self.service_host)
            self.cache = record[:2]
            self.cache_start_time = time.time()
            self.cache_ttl = record[2]

    	return self.cache

    class _HTTPConnection(httplib.HTTPConnection):
        def connect(self):
            solver = DnsServiceResolver(self.host)
    	print solver.resolve()
            self.sock = socket.create_connection(solver.resolve(), self.timeout)

    class _HTTPSConnection(httplib.HTTPConnection):
        def connect(self):
            solver = DnsServiceResolver(self.host)
            sock = socket.create_connection(solver.resolve(), self.timeout)
            self.sock = ssl.wrap_socket(sock, self.key_file, self.cert_file)

    class _HTTPHandler(urllib2.HTTPHandler):
        def http_open(self, req):
            return self.do_open(_HTTPConnection, req)

    class _HTTPSHandler(urllib2.HTTPSHandler):
        def https_open(self, req):
            return self.do_open(_HTTPSConnection, req)

    _opener = urllib2.build_opener(_HTTPHandler, _HTTPSHandler)
    urllib2.install_opener(_opener)

# 最后我只能说这玩意吊炸天

把host配置的问题解决掉了，就可以发布对运行环境完全无依赖的代码和配置了。发布程序就变成了发布container。

