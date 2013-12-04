title: SkyDNS 动手玩
slug: skydns-get-started
date: 2013-11-23 23:00
tags: 后台开发, 架构
category: 后台开发
author: 火山

### 资源

github: https://github.com/skynetservices/skydns

- 编译：go build .
- 启动脚本：sudo /home/ken/bin/skydns --domain=kuaiwan.local --data=/home/ken/bin/.skydns_data

### 增加DNS解析服务器

	vim /etc/resolv.conf
	增加: nameserver localhost

### 增加DNS记录

`curl -X PUT -L http://localhost:8080/skydns/services/1001 -d '{"Name":"TestService","Version":"1.0.0","Environment":"Production","Region":"East","Host":"localhost","Port":80,"TTL":4000}'`

`curl -X PUT -L http://localhost:8080/skydns/services/1002 -d '{"Name":"TestService","Version":"1.0.0","Environment":"Production","Region":"East","Host":"192.168.60.61","Port":80,"TTL":4000}'`

### 测试

`dig @localhost testservice.production.kuaiwan.local SRV`

本地其他客户端可能发送DNS请求给我们的DNS服务器：

    2013/11/23 22:46:59 Received DNS Request for "localdomain." from "127.0.0.1:35947"
    2013/11/23 22:47:01 Received DNS Request for "ken-virtualbox.localdomain." from "127.0.0.1:52030"
    2013/11/23 22:47:03 Received DNS Request for "gc._msdcs.localdomain." from "127.0.0.1:33695"
    

### python客户端使用

内置dns库

    :::python
    import dns 
    ans = dns.resolver.default_resolver.query('testservice.production.kuaiwan.local.', 'SRV', 1, True)
	print ans.expiration
    print ans.rrset
    print ans.response.answer[0]
    testservice.production.kuaiwan.local. 929 IN SRV 10 50 80 localhost.
    testservice.production.kuaiwan.local. 929 IN SRV 10 50 80 192.168.60.61.

注意事项：

	- SkyDNS是TCP服务器
	- rdclass的取值是1，表示IN
	- rdtype的取值是SRV，取值是33

使用pyDNS

    :::python
    import DNS
    DNS.ParseResolvConf()
    srv_req = DNS.Request(qtype='srv')
    srv_result = srv_req.req('testservice.production.kuaiwan.local')
    srv_result.answers
    [{'name': 'testservice.production.kuaiwan.local', 'data': (10, 100, 80, 'localhost'), 'typename': 'SRV', 'classstr': 'IN', 'ttl': 517, 'type': 33, 'class': 1, 'rdlength': 21}]

### 应用

可以考虑改造Rest接口的客户端模块，让其支持解析ServiceName并获取服务IP列表。现在python客户端测试看来默认没有缓存，可能需要自己实现DNS结果缓存机制。

FINAL TEST
