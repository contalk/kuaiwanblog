Title: 快币系统冻结接口调优
Date: 2014-01-23 18:30
Category: 后台开发
Tags: program optimization, Mysql, Redis
slug: somethings-about-optimizing-balance-freeze-interface
Author: leon


从1月9日到昨天上午，接近两周的时间都在集中精力解决快币系统冻结接口响应时间过长的问题。从开始处理、不断尝试到今天上午终于告一段落。几经波折，最后解决方式非常简单，甚至有点让我哭笑不得。

# 1. What

如前所述，问题在于快币的冻结接口响应很慢，这个接口是提供给云帆流量矿石交易系统的。到底有多慢呢？一个每秒请求不到 1 次的 HTTP API，响应时间经常出现 3s、4s，平均时间约 1s 的性能表现已经说明了一切。这个接口的业务逻辑和数据库操作都不复杂，而且同一个系统上其他请求量、业务逻辑和数据操作都接近的接口，都没有发生类似的异常。

# 2. How

## Step 1. 看代码

第一步当然是从代码入手。这个接口主要做了这样的事情：

1. 参数验证（包括类型转换、`APP_ID`许可验证、订单超时验证、签名验证）
2. 给当前用户“加锁”（确保同一个用户的数据只有一个进程在写，用 Redis 实现，不阻塞）
3. 检查当前用户的快币账户（账户不存在或余额不足都不能完成请求）
4. 检查当前订单冻结记录（只有未冻结的订单才能继续）
5. 处理业务（修改账户、初始化数据库记录、写磁盘日志）
6. 向数据库提交事务
7. 给当前用户“解锁”
8. 返回

从代码上几乎看不出任何问题，条理清晰，没有复杂的运算，为了简单处理，加锁的实现也只是直接返回错误，而不是让进程阻塞。

在这些步骤当中，有一次Redis操作，几次mysql读写，除此之外都是标准库调用。

其中用Redis对用户下单频率做限制的实现是：

    :::python
    def get_freeze_limit(user_id):
        """ 限制下单的频率 """
        key = "LOCK:USER:%s" % user_id
        return client.get(key)    

    def set_freeze_limit(user_id):
        key = "LOCK:USER:%s" % user_id
        client.setex(key, user_id, settings.FREEZE_TIMEOUT)

这个简单的实现虽然在业务逻辑上有缺陷，但也看不出有什么性能问题。我们在其他应用也大量使用Redis来加快查询和减轻数据库的负担，谁还会去怀疑内存操作的速度。

另一个是数据库操作，在一个事务里会执行两个 `insert` 和一个 `update`，在这之前的快币账户检查和订单检查分别有一次 `select`，也就是存在磁盘 I/O。既然代码没问题，就把注意力转移到了数据库上面。

## Step 2. 内网测试

问题暂时定位在数据库上面，但在发现问题之前还是不能盲目操作，而且也无从下手。所以先在内网搭建了一个和线上类似的环境先做一下测试。说“类似的环境”是因为很难（至少对于我）完全模拟线上的真实环境，包括应用请求量、数据库实例的 query 数（线上同一个实例还有其他充值相关的数据库）。

把线上的数据库实例 `my_pay` 配置文件和快币的数据库 `kuaibi` 在内网一台比较空闲的数据库服务器（硬件配置接近）配置好，应用和 Redis 都跑在本地开发机（Ubuntu 12.04），gunicorn worker 数量相同，Redis 使用默认配置。用 gevent 并发请求冻结接口，为了弥补其他接口和应用的影响，请求数和并发量都远高于线上真实环境。

内网测试的表现让人很满意，在 `count_ps` 为两三百的水平（线上连 1 都不到）下，每个请求的平均响应时间维持在 21ms 左右，最高不超过 23ms。后来还对整个请求做了 split，统计了每个步骤的执行时间。结果也在意料之中，主要时间消耗在数据库操作上面，每个 select 大概 3ms，插入/更新操作需要 13ms 左右。这个数据和线上的表现，差的是一个数量级。

## Step 3. 修改实例配置

内网测试没有发现问题，几乎没什么进展，注意力回到线上环境。之前购买的 [Qmonitor服务](http://www.woqutech.com/?page_id=5) 开始发挥作用。Qmonitor 可以对数据库服务器和实例做多种粒度的性能监控和统计，还能生成报表。在 Qmonitor 的管理后台可以很方便地查看各指标的性能曲线，还能把不同的数据库实例放在一起做对比。

为了排除硬件（磁盘、内存、OS等）的差异，我们把 `my_pay` 和同台服务器上的另一个用于快玩硬件用户的实例 `my_user` 进行了对比（实际上还有其他实例）。结果发现在 `my_user` 查询更多的情况下， *InnoDB row read* 这个指标的差异非常明显，`my_pay` 的这个值在 5w-10w 行/秒 之间，有时候还更高，而 `my_user` 基本上都在 200 以内。与此对应的是 `my_pay` 的 *InnoDB disk traffic* （20-30M）, *InnoDB read IOPS* （700-1000）, *InnoDB disk read* （基本与 InnoDB disk traffic 一致） 等几个指标都比 `my_user` 的数值高很多。

- InnoDB row read：InnoDB 每秒读取的行数
- InnoDB disk traffic：InnoDB 每秒的磁盘 I/O 流量
- InnoDB read IOPS：InnoDB 每秒进行读 I/O 的数目，每次可能读取多个页
- InnoDB disk read：InnoDB 每秒读取物理数据文件的大小

通过观察和对比，的确发现了一些问题（或特点）：

1. `my_pay` 的 读查询 >> 写查询，从 query 类型和业务逻辑（快币查询接口很繁忙）都可以很容易确认这一点，而且将来读查询还会增加的更快
2. `my_pay` 的查询需要大量从磁盘读取内容，而不是缓冲区

进一步对比 `my_pay` 和 `my_user` 的实例配置，发现了几个参数不一样：

    :::text
    +----------------------------------------------------------------------+
    |         Configuration          |     my_pay       |   my_user        |
    +--------------------------------+------------------+------------------+
    | innodb_buffer_pool_size        | 402653184 (384M) | 2147483648 (2G)  |
    | innodb_log_file_size           | 104857600 (100M) | 268435456 (256M) |
    | innodb_flush_log_at_trx_commit | 1                | 2                |
    | sync_binlog                    | 1                | 5                |
    +--------------------------------+------------------+------------------+
    
- `innodb_buffer_pool_size`：InnoDB用来缓存表数据和索引，值越大，需要的磁盘I/O越少。默认8M，对于专用的数据库服务器，最大可以设为物理内存80%，过大可能导致操作系统内存分页。缓冲区初始化所需时间和缓冲区大小正相关。

- `innodb_log_file_size`：每个log group里日志文件的大小，默认值5M，理想值域 [1M, innodb_buffer_pool_size/n]，其中 n 表示每个log group中日志文件的数量，即 `innodb_log_files_in_group`，推荐值是2. `innodb_log_file_size` 越大，越节省磁盘 I/O，不过如果太大在崩溃后recovery就更慢。

- `innodb_flush_log_at_trx_commit`：InnoDB事务提交后的日志写入频率。`1` 表示每次都写入日志文件；`2` 表示写入操作系统缓存，然后mysql自动每秒flush到磁盘；`0` 表示不写日志。

- `sync_binlog`：mysql把它的binary log同步到磁盘的频率。如果autocommit开启，每个语句都写一次binary log，否则每次事务写一次。默认值是 `0`，不主动同步，而依赖操作系统本身不定期把文件内容flush到磁盘。设为 `1` 最安全，在每个语句或事务后同步一次binary log，即使在崩溃时也最多丢失一个语句或事务的日志，但同时也最慢（除非磁盘有battery-backed cache）。

对于事务繁忙的数据库来说，`innodb_flush_log_at_trx_commit` 和 `sync_binlog` 两个参数的设置对数据库的写入性能会有很大影响。由于充值系统对安全要求很高，当前两个参数的配置都是最慢而最安全的取值。而且如前所述，这个实例的写入操作并不频繁。所以主要锁定了 `innodb_buffer_pool_size` 和 `innodb_log_file_size` 两个配置项。

那么如何调整呢？

`innodb_buffer_pool_size` 的调整主要参考了两篇材料：

[1]: [Choosing innodb_buffer_pool_size](http://www.mysqlperformanceblog.com/2007/11/03/choosing-innodb_buffer_pool_size/)

[2]: [How large should be mysql innodb_buffer_pool_size](http://dba.stackexchange.com/q/27328/)

如前所述，`innodb_buffer_pool_size` 是 InnoDB 用来缓存表数据和索引的缓冲区大小。所以这个值的设定取决于表数据大小和索引大小（`tablespace`），这也是上面两篇文章都认同的。需要注意的是，InnoDB 实际上会分配比这个值多 10% 的内存。

上面两篇文章的区别在于，[1] 认为比 `tablespace` 大 10%，[2] 认为大 60%. [2] 给出了一个 sql语句来帮助计算取值：

    :::sql
    SELECT CEILING(Total_InnoDB_Bytes*1.6/POWER(1024,3)) RIBPS FROM
    (SELECT SUM(data_length+index_length) Total_InnoDB_Bytes
    FROM information_schema.tables WHERE engine='InnoDB') A;

这个查询的结果是 5G，把系数改成 1.1 得到的结果是 4G. 综合考虑了一下最后选择 4G.

`innodb_log_file_size` 的值可以通过观察 `Innodb_os_log_written` 估算出来，在服务器上执行sql：

    :::sql
    show status like 'Innodb_os_log_written'; select sleep(60); show status like 'Innodb_os_log_written';

两次结果的差值就是 1 分钟内日志文件的写入大小。多次执行，得到大约每分钟 0.5M，假设高峰期是4倍，取 2M，根据 [3] 的观点，假如日志保存 1h，日志文件大小 2M*60=120M，取 128M，一共两个日志文件，每个 64M 就足够了。所以当前 100M 取值是完全可以的。如果还担心不够，可以设大一点，如 256M.

[3]: [How to calculate a good InnoDB log file size](http://www.mysqlperformanceblog.com/2008/11/21/how-to-calculate-a-good-innodb-log-file-size/)

所以最后给出的解决方案，其实只是调整了一个参数：

    :::cnf
    innodb_buffer_pool_size = 4G

做了这个调整之后，再在 Qmonitor 和 [Graphite](http://graphite.wikidot.com/) 上观察数据库和应用性能监控曲线，发现 InnoDB read IOPS 从原来的 700-1000 下降到了 接近 0，InnoDB disk read 和 InnoDB disk traffic 也都有明显下降。 但是 InnoDB row read 和冻结接口的响应时间基本没什么变化。

所以问题仍然没有解决。

## Step 4. 清数据、加索引、优化查询

上一步的调整虽然对解决问题没有帮助，但对于减少数据库磁盘I/O还是效果显著。其实上面列出的四项 InnoDB 指标，InnoDB row read 和其他 3 项分属两类，前者是扫描行数，还是在数据库层面，而后者是I/O层面。InnoDB row read 很大很可能有这些原因：

1. 某个表数据量大
2. 有复杂的查询，如 join、group by、sort 等
3. 缺少索引

通过 Qmonitor 还发现一个很奇怪的现象，`my_pay` 的 InnoDB row read 曲线有很明显的规律：

- 处于高位运行（5w/s），说明普通查询中有低效查询
- 每 5 分钟出现一次高峰（10w/s），说明有 5 分钟一次的定时大开销的查询。
- 凌晨没有下降，说明查询不是来自用户

打开 crontab 定时任务列表，很容易找到和这个实例有关的任务和查询（之前没有开启慢查询）发现有一个每分钟执行的任务，还有一个每5分钟执行的任务。第一个任务会先执行一个join，然后再做一个 `update_time` 的范围查询；第二个任务也是 `update_time` 的范围查询。后来还发现一个每5分钟被请求一次的接口，这个接口做了 `update_time` 的范围查询，然后在 `create_time` 字段上对结果排序。这些查询都来自充值服务，数据库（`charge`）用了同一个实例。

这个数据库上有2个表，其中一个接近 380w 条记录，另一个 200w。而且，`update_time` 没有索引。所以，这就是导致 InnoDB row read 很高的罪魁祸首。定位了问题就很容易解决，在做好备份和权衡后，对这两个表做了一些数据清理，也把 `update_time` 字段，还有一些其他必要的字段一并加了索引。

删除据过程很顺利，在停服务加索引的时候遇到了等待锁的问题。由此还查出了代码里有些地方没有正确关闭数据库连接，也一并优化了。做完这些操作，对 InnoDB row read 的影响很明显，从原来的 5w-10w 下降到 1k 的水平。

但是，冻结接口的性能依然很慢。

## Step 5. 读写分离

快币查询接口的请求量比较大，在高峰的时候每秒有300多个请求。在我清数据的同时，遥遥也把这个接口的查询转移到了从库，做了读写分离。分离过后，对查询接口本身的性能倒是有明显提高，从原来的 6ms 降到了 3ms 左右。

但是，冻结接口的性能依然很慢。

## Step 5. 数据库迁移

做了这么多操作都没有成效，问题依然悬而未决，而且似乎根本没有接近问题的根本。这时候还有 2 个选择：把快币的数据库独立出来；对线上的冻结接口做细粒度的时间统计。

从数据量来看，快币的数据库用一个单独的实例完全没有必要，而且上面的优化也把这个实例的性能提高了不少。但从业务来看，这样做是很有必要的。快币是一个核心应用，而且对安全性有很高的要求。之前的优化和数据清理，其实都是因为其他应用的数据库带来的影响。把快币的数据库独立出来，可以保证受到最小的外部干扰和连带影响。

数据库迁移其实比较简单，需要注意的是确保数据不丢失，应用能顺利迁移。整个流程大致如此：

1. 建好新的实例，包括主从同步
2. 停止所有相关服务
3. 导出整个数据库，为了确保切换后及时暴露问题，可以把老的数据库里的表全部重命名
4. 导入主库，检查老库、新主库、新从库的数据是否一致
5. 修改应用配置，重新开启服务
6. 检查服务是否正常，数据是否一致

在陈果的帮助下，先在内网学习了一遍主从配置。其他步骤自己都很熟悉，也在内网演练了一遍。22日早上 7 点开始执行，原本简单的一次迁移因为人为疏忽出了差错。我在把数据导入新库的时候，不慎导入了从库，而且因此破坏了主从同步。Drop 掉数据，修复主从同步，再重新导入，终于迁移完成。

执行这个操作时，只有我一个人，早上精神状态也没有达到最佳。不过这些都不重要，都只是借口。对下面这几个教训深有体会：

1. 类似的操作最好有 2 个人一起完成，可以相互提醒
2. 做再多的准备都不为过，永远想好失败的应对方案，留好退路
3. 可能出错的地方就一定会出错

## Step 6. 做细粒度的时间统计

这个其实在内网做过，没能发现问题。但是在线上环境做一下统计，就可以很直接就确定到底是哪个步骤出了问题。这个的统计其实可以一开始就做，但需要插入一些和业务逻辑无关的代码，弄乱代码结构。现在想想，与找到问题根本原因相比，其实这个并不重要。如果早做了这一步，很快就能解决问题。

在这段时间内，把给用户加锁的代码完善了一下：

    :::python
    def acquire_lock(user_id):
        """ 请求锁 """
        lock_key = "LOCK:USER:%s" % user_id
        success = True
        expire_time = time.time() + settings.FREEZE_TIMEOUT
    
        acquired = client.setnx(lock_key, "%s" % expire_time)
    
        # 锁已经存在
        if not acquired:
            # 获取上一次锁的有效期
            old_expire = client.get(lock_key)
            # 发现超时，可能是上一个获取锁的进程发生异常，没有正常解锁
            if decimal.Decimal(old_expire) < decimal.Decimal("%s" % time.time()):
                # 再次尝试获取锁
                # 获取锁当前的有效期
                now_expire = client.getset(lock_key, expire_time)
    
                # 没有超时，在这瞬间被其他进程捷足先登
                if decimal.Decimal(now_expire) > decimal.Decimal("%s" % time.time()):
                    success = False
            else:
                success = False
    
        return success
    
    
    def release_lock(user_id):
        """ 释放锁 """
        lock_key = "LOCK:USER:%s" % user_id
        client.delete(lock_key)

为了解决死锁的问题，比原来的加锁逻辑更复杂一些。除此之外，冻结接口的其他代码都没变化。21 日晚上更新了代码，在 22 日看监控图发现了一个“惊人”的事实：整个接口的响应时间和请求锁的时间完美同步（最长的甚至有三四秒），相比之下，数据库操作消耗的时间都很短（30ms 以内）。

这个结果说明，我们之前都在一个错误的方向努力，罪魁祸首从一开始就被忽略。

## Step 7. Redis 才是元凶

因为用Redis而导致接口很慢，原因只有一个，就是使用的方法不对。

找到对应的实例，在配置文件里发现了这些扎眼的配置：

    :::cnf
    appendonly yes
    appendfsync always

*Append Only File (AOF)* 是 Redis 用于持久化和恢复的日志文件，类似于 Mysql 的 binlog. `appendonly` 选项是AOF的开关。在开启AOF的时候，`appendfsync` 决定了日志文件 flush 到磁盘的频率，类似于 InnoDB 里的 `innodb_flush_log_at_trx_commit`. `appendfsync` 有三种取值：

- `always`： 每条命令都添加到 AOF，如 [文档](http://redis-docs.readthedocs.org/en/latest/AppendOnlyFileHowto.html#how-durable-is-the-append-only-file) 所述 
    > Very very slow, very safe.
- `everysec`: 每秒添加一次，最多可能丢失 1 秒的日志
- `no`：不主动刷写，由操作系统调度，速度最快，也最不安全

而我们用的就是 "Very very slow" 的配置，所以导致冻结接口也 very very slow.

关掉 `appendonly` 再查看接口表现，效果立竿见影，请求时间从原来的 1s 级下降到了 30ms 以内，而且很稳定。

问题就这么解决了，终于送一口气。同时哭笑不得。