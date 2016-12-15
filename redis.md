### 1 什么是redis 

redis是一个key-value存储系统。和Memcached类似，它支持存储的value类型相对更多，包括string\(字符串\)、list\(链表\)、set\(集合\)和zset\(有序集合\)。这些数据类型都支持push/pop、add/remove及取交集并集和差集及更丰富的操作，而且这些操作都是原子性的。在此基础上，redis支持各种不同方式的排序。与memcached一样，为了保证效率，数据都是缓存在内存中。区别的是redis会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件，并且在此基础上实现了master-slave\(主从\)同步。  

## 2 性能怎么样 

Redis是一个高性能的key-value内存数据库。官方性能测试结果： set操作每秒110000次，get操作每秒81000次。 

## 3 可不可以存对象 

和Memcached类似，它支持存储的value类型相对更多，包括string\(字符串\)、list\(链表\)、set\(集合\)和zset\(有序集合\)。这些数据类型都支持push/pop、add/remove及取交集并集和差集及更丰富的操作。  

## 4 Redis与memcache的最大区别 

Replication（树形） 

data types（String、Lists、Sorted Sets、Hashes） persistence \(snapshot、aof\) 

很多开发者都认为Redis不可能比Memcached快，Memcached完全基于内存，而Redis具有持久化保存特性，即使是异步的，Redis也不可能比Memcached快。但是测试结果基本是Redis占绝对优势。一直在思考这个原因，目前想到的原因有这几方面。 

Libevent。和Memcached不同，Redis并没有选择libevent。Libevent为了迎合通用性造成代码庞大\(目前Redis代码还不到libevent的1/3\)及牺牲了在特定平台的不少性能。Redis用libevent中两个文件修改实现了自己的epoll event loop\(4\)。业界不少开发者也建议Redis使用另外一个libevent高性能替代libev，但是作者还是坚持Redis应该小巧并去依赖的思路。一个印象深刻的细节是编译Redis之前并不需要执行./configure。 

CAS问题。CAS是Memcached中比较方便的一种防止竞争修改资源的方法。CAS实现需要为每个cache key设置一个隐藏的cas token，cas相当value版本号，每次set会token需要递增，因此带来CPU和内存的双重开销，虽然这些开销很小，但是到单机10G+ cache以及QPS上万之后这些开销就会给双方相对带来一些细微性能差别\(5\)。 

## 5单台Redis的存放数据必须比物理内存小 

Redis的数据全部放在内存带来了高速的性能，但是也带来一些不合理之处。比如一个中型网站有100万注册用户，如果这些资料要用Redis来存储，内存的容量必须能够容纳这100万用户。但是业务实际情况是100万用户只有5万活跃用户，1周来访问过1次的也只有15万用户，因此全部100万用户的数据都放在内存有不合理之处，RAM需要为冷数据买单。 这跟操作系统非常相似，操作系统所有应用访问的数据都在内存，但是如果物理内存容纳不下新的数据，操作系统会智能将部分长期没有访问的数据交换到磁盘，为新的应用留出空间。现代操作系统给应用提供的并不是物理内存，而是虚拟内存\(Virtual Memory\)的概念。 基于相同的考虑，Redis 2.0也增加了VM特性。让Redis数据容量突破了物理内存的限制。并实现了数据冷热分离。  

## 6 Redis的VM实现是重复造轮子 

Redis的VM依照之前的epoll实现思路依旧是自己实现。但是在前面操作系统的介绍提到OS也可以自动帮程序实现冷热数据分离，Redis只需要OS申请一块大内存，OS会自动将热数据放入物理内存，冷数据交换到硬盘，另外一个知名的“理解了现代操作系统\(3\)”的Varnish就是这样实现，也取得了非常成功的效果。 

作者antirez在解释为什么要自己实现VM中提到几个原因\(6\)。主要OS的VM换入换出是基于Page概念，比如OS VM1个Page是4K, 4K中只要还有一个元素即使只有1个字节被访问，这个页也不会被SWAP, 换入也同样道理，读到一个字节可能会换入4K无用的内存。而Redis自己实现则可以达到控制换入的粒度。另外访问操作系统SWAP内存区域时block进程，也是导致Redis要自己实现VM原因之一。 

## 7 用get/set方式使用Redis 

作为一个key value存在，很多开发者自然的使用set/get方式来使用Redis，实际上这并不是最优化的使用方法。尤其在未启用VM情况下，Redis全部数据需要放入内存，节约内存尤其重要。 

假如一个key-value单元需要最小占用512字节，即使只存一个字节也占了512字节。这时候就有一个设计模式，可以把key复用，几个key-value放入一个key中，value再作为一个set存入，这样同样512字节就会存放10-100倍的容量。 

这就是为了节约内存，建议使用hashset而不是set/get的方式来使用Redis 

## 8使用aof代替snapshot 

Redis有两种存储方式，默认是snapshot方式，实现方法是定时将内存的快照\(snapshot\)持久化到硬盘，这种方法缺点是持久化之后如果出现crash则会丢失一段数据。因此在完美主义者的推动下作者增加了aof方式。aof即append only mode，在写入内存数据的同时将操作命令保存到日志文件，在一个并发更改上万的系统中，命令日志是一个非常庞大的数据，管理维护成本非常高，恢复重建时间会非常长，这样导致失去aof高可用性本意。另外更重要的是Redis是一个内存数据结构模型，所有的优势都是建立在对内存复杂数据结构高效的原子操作上，这样就看出aof是一个非常不协调的部分。 

其实aof目的主要是数据可靠性及高可用性，在Redis中有另外一种方法来达到目的：Replication。由于Redis的高性能，复制基本没有延迟。这样达到了防止单点故障及实现了高可用。    









































































󰀀   zrank\(key, member\) ：返回名称为key的zset（元素已按score从小到大排序）中member元素的rank（即index，从0开始），若没有member元素，返回“nil” 

󰀀   zrevrank\(key, member\) ：返回名称为key的zset（元素已按score从大到小排序）中member元素的rank（即index，从0开始），若没有member元素，返回“nil” 

󰀀   zrange\(key, start, end\)：返回名称为key的zset（元素已按score从小到大排序）中的index从start到end的所有元素 

󰀀   zrevrange\(key, start, end\)：返回名称为key的zset（元素已按score从大到小排序）中的index从start到end的所有元素 

󰀀   zrangebyscore\(key, min, max\)：返回名称为key的zset中score &gt;= min且score &lt;= max的所有元素 

󰀀   zcard\(key\)：返回名称为key的zset的基数 

󰀀   zscore\(key, element\)：返回名称为key的zset中元素element的score 󰀀   zremrangebyrank\(key, min, max\)：删除名称为key的zset中rank &gt;= min且rank &lt;= max的所有元素 

󰀀   zremrangebyscore\(key, min, max\) ：删除名称为key的zset中score &gt;= min且score &lt;= max的所有元素 

󰀀   zunionstore / zinterstore\(dstkeyN, key1,„,keyN, WEIGHTS w1,„wN, AGGREGATE SUM\|MIN\|MAX\)：对N个zset求并集和交集，并将最后的集合保存在dstkeyN中。对于集合中每一个元素的score，在进行AGGREGATE运算前，都要乘以对于的WEIGHT参数。如果没有提供WEIGHT，默认为1。默认的AGGREGATE是SUM，即结果集合中元素的score是所有集合对应元素进行SUM运算的值，而MIN和MAX是指，结果集合中元素的score是所有集合对应元素中最小值和最大值。 

对Hash操作的命令 

󰀀   hset\(key, field, value\)：向名称为key的hash中添加元素field&lt;—&gt;value 

󰀀   hget\(key, field\)：返回名称为key的hash中field对应的value 



















󰀀   hmget\(key, field1, „,field N\)：返回名称为key的hash中field i对应的value 

󰀀   hmset\(key, field1, value1,„,field N, value N\)：向名称为key的hash中添加元素field i&lt;—&gt;value i 

󰀀   hincrby\(key, field, integer\)：将名称为key的hash中field的value增加integer 

󰀀   hexists\(key, field\)：名称为key的hash中是否存在键为field的域 󰀀   hdel\(key, field\)：删除名称为key的hash中键为field的域 󰀀   hlen\(key\)：返回名称为key的hash中元素个数 󰀀   hkeys\(key\)：返回名称为key的hash中所有键 

󰀀   hvals\(key\)：返回名称为key的hash中所有键对应的value 

󰀀   hgetall\(key\)：返回名称为key的hash中所有的键（field）及其对应的value 

持久化 

󰀀   save：将数据同步保存到磁盘 󰀀   bgsave：将数据异步保存到磁盘 

󰀀   lastsave：返回上次成功将数据保存到磁盘的Unix时戳 󰀀   shundown：将数据同步保存到磁盘，然后关闭服务 

远程服务控制 

󰀀   info：提供服务器的信息和统计 󰀀   monitor：实时转储收到的请求 󰀀   slaveof：改变复制策略设置 󰀀   config：在运行时配置Redis服务器 



















   

在192.168.134.96 上安装了redis的master    

在192.168.134.97  上安装了slave 绑定了192.168.134.96 









































































󰀀   zrank\(key, member\) ：返回名称为key的zset（元素已按score从小到大排序）中member元素的rank（即index，从0开始），若没有member元素，返回“nil” 

󰀀   zrevrank\(key, member\) ：返回名称为key的zset（元素已按score从大到小排序）中member元素的rank（即index，从0开始），若没有member元素，返回“nil” 

󰀀   zrange\(key, start, end\)：返回名称为key的zset（元素已按score从小到大排序）中的index从start到end的所有元素 

󰀀   zrevrange\(key, start, end\)：返回名称为key的zset（元素已按score从大到小排序）中的index从start到end的所有元素 

󰀀   zrangebyscore\(key, min, max\)：返回名称为key的zset中score &gt;= min且score &lt;= max的所有元素 

󰀀   zcard\(key\)：返回名称为key的zset的基数 

󰀀   zscore\(key, element\)：返回名称为key的zset中元素element的score 󰀀   zremrangebyrank\(key, min, max\)：删除名称为key的zset中rank &gt;= min且rank &lt;= max的所有元素 

󰀀   zremrangebyscore\(key, min, max\) ：删除名称为key的zset中score &gt;= min且score &lt;= max的所有元素 

󰀀   zunionstore / zinterstore\(dstkeyN, key1,„,keyN, WEIGHTS w1,„wN, AGGREGATE SUM\|MIN\|MAX\)：对N个zset求并集和交集，并将最后的集合保存在dstkeyN中。对于集合中每一个元素的score，在进行AGGREGATE运算前，都要乘以对于的WEIGHT参数。如果没有提供WEIGHT，默认为1。默认的AGGREGATE是SUM，即结果集合中元素的score是所有集合对应元素进行SUM运算的值，而MIN和MAX是指，结果集合中元素的score是所有集合对应元素中最小值和最大值。 

对Hash操作的命令 

󰀀   hset\(key, field, value\)：向名称为key的hash中添加元素field&lt;—&gt;value 

󰀀   hget\(key, field\)：返回名称为key的hash中field对应的value 



















󰀀   hmget\(key, field1, „,field N\)：返回名称为key的hash中field i对应的value 

󰀀   hmset\(key, field1, value1,„,field N, value N\)：向名称为key的hash中添加元素field i&lt;—&gt;value i 

󰀀   hincrby\(key, field, integer\)：将名称为key的hash中field的value增加integer 

󰀀   hexists\(key, field\)：名称为key的hash中是否存在键为field的域 󰀀   hdel\(key, field\)：删除名称为key的hash中键为field的域 󰀀   hlen\(key\)：返回名称为key的hash中元素个数 󰀀   hkeys\(key\)：返回名称为key的hash中所有键 

󰀀   hvals\(key\)：返回名称为key的hash中所有键对应的value 

󰀀   hgetall\(key\)：返回名称为key的hash中所有的键（field）及其对应的value 

持久化 

󰀀   save：将数据同步保存到磁盘 󰀀   bgsave：将数据异步保存到磁盘 

󰀀   lastsave：返回上次成功将数据保存到磁盘的Unix时戳 󰀀   shundown：将数据同步保存到磁盘，然后关闭服务 

远程服务控制 

󰀀   info：提供服务器的信息和统计 󰀀   monitor：实时转储收到的请求 󰀀   slaveof：改变复制策略设置 󰀀   config：在运行时配置Redis服务器 



















   

在192.168.134.96 上安装了redis的master    

在192.168.134.97  上安装了slave 绑定了192.168.134.96 

11 redis事务 

redis对事务的支持目前还比较简单。redis只能保证一个client发起的事务中的命令可以连续的执行，而中间不会插入其他client的命令。由于redis是单线程来处理所有client的请求的所以做到这点是很容易的。一般情况下redis在接受到一个client发来的命令后会立即处理并返回处理结果，但是当一个client在一个连接中发出multi命令有，这个连接会进入一个事务上下文，该连接后续的命令并不是立即执行，而是先放到一个队列中。当从此连接受到exec命令后，redis会顺序的执行队列中的所有命令。并将所有命令的运行结果打包到一起返回给client.然后此连接就结束事务上下文。下面可以看一个例子 

redis&gt; multi OK 

redis&gt;incr a QUEUED redis&gt;incrb QUEUED redis&gt; exec 1. \(integer\) 1 2. \(integer\) 1 

从这个例子我们可以看到incr a ,incr b命令发出后并没执行而是被放到了队列中。调用exec后俩个命令被连续的执行，最后返回的是两条命令执行后的结果 我们可以调用discard命令来取消一个事务。接着上面例子 

redis&gt; multi OK 

redis&gt;incr a QUEUED redis&gt;incrb QUEUED 

redis&gt; discard OK 

redis&gt; get a "1" 

redis&gt; get b "1" 

可以发现这次incr a incr b都没被执行。discard命令其实就是清空事务的命令队列并退出事务上下文。 



















  虽说redis事务在本质上也相当于序列化隔离级别的了。但是由于事务上下文的命令只排队并不立即执行，所以事务中的写操作不能依赖事务中的读操作结果。看下面例子 

redis&gt; multi OK 

redis&gt; get a QUEUED redis&gt; get b QUEUED redis&gt; exec 1. "1" 2. "1" 

发现问题了吧。假如我们想用事务实现incr操作怎么办？可以这样做吗？ 

redis&gt; get a "1" 

redis&gt; multi OK 

redis&gt; set a 2 QUEUED redis&gt; exec 1. OK 

redis&gt; get a， "2" 

结论很明显这样是不行的。这样和 get a 然后直接set a是没区别的。很明显由于get a 和set a并不能保证两个命令是连续执行的\(get操作不在事务上下文中\)。很可能有两个client同时做这个操作。结果我们期望是加两次a从原来的1变成3. 但是很有可能两个client的get a，取到都是1，造成最终加两次结果却是2。主要问题我们没有对共享资源a的访问进行任何的同步 

也就是说redis没提供任何的加锁机制来同步对a的访问。 

还好redis 2.1后添加了watch命令，可以用来实现乐观锁。看个正确实现incr命令的例子,只是在前面加了watch a 

redis&gt; watch a OK 

redis&gt; get a "1" 

redis&gt; multi OK 

redis&gt; set a 2 QUEUED redis&gt; exec 1. OK 

redis&gt; get a， "2" 

watch 命令会监视给定的key,当exec时候如果监视的key从调用watch后发生过变化，则整个事务会失败。也可以调用watch多次监视多个key.这样就可以对指定的key加乐观锁了。注意watch的key是对整个连接有效的，事务也一样。如果连接断开，监视和事务都会被自动清除。当然了exec,discard,unwatch命令都会清除连接中的所有监视. redis的事务实现是如此简单，当然会存在一些问题。第一个问题是redis只能保证事务的每个命令连续执行，但是如果事务中的一个命令失败了，并不回滚其他命令，比如使用的命令类型不匹配。 

redis&gt; set a 5 OK 

redis&gt;lpushb 5 \(integer\) 1 redis&gt; set c 5 OK 

redis&gt; multi OK 

redis&gt;incr a QUEUED redis&gt;incrb QUEUED redis&gt;incr c QUEUED redis&gt; exec 1. \(integer\) 6 

2. \(error\) ERR Operation against a key holding the wrong kind of value 3. \(integer\) 6 

可以看到虽然incr b失败了，但是其他两个命令还是执行了。 最后一个十分罕见的问题是当事务的执行过程中，如果redis意外的挂了。很遗憾只有部分命令执行了，后面的也就被丢弃了。当然如果我们使用的append-only file方式持久化，redis会用单个write操作写入整个事务内容。即是是这种方式还是有可能只部分写入了事务到磁盘。发生部分写入事务的情况下，redis重启时会检测到这种情况，然后失败退出。可以使用redis-check-aof工具进行修复，修复会删除部分写入的事务内容。修复完后就能够重新启动了。

## 12 redis的持久化 

redis是一个支持持久化的内存数据库，也就是说redis需要经常将内存中的数据同步到磁盘来保证持久化。redis支持两种持久化方式，一种是 Snapshotting（快照）也是默认方式，另一种是Append-only file（缩写aof）的方式。下面分别介绍 

Snapshotting 

       快照是默认的持久化方式。这种方式是就是将内存中数据以快照的方式写入到二进制文件中,默认的文件名为dump.rdb。可以通过配置设置自动做快照持久化的方式。我们可以配置redis在n秒内如果超过m个key被修改就自动做快照，下面是默认的快照保存配置 

save 900 1  \#900秒内如果超过1个key被修改，则发起快照保存 save 300 10 \#300秒内容如超过10个key被修改，则发起快照保存 



















save 60 10000 

下面介绍详细的快照保存过程 

1.redis调用fork,现在有了子进程和父进程。 

2. 父进程继续处理client请求，子进程负责将内存内容写入到临时文件。由于os的写时复制机制（copy on write\)父子进程会共享相同的物理页面，当父进程处理写请求时os会为父进程要修改的页面创建副本，而不是写共享的页面。所以子进程的地址空间内的数据是fork时刻整个数据库的一个快照。 

3.当子进程将快照写入临时文件完毕后，用临时文件替换原来的快照文件，然后子进程退出。 client 也可以使用save或者bgsave命令通知redis做一次快照持久化。save操作是在主线程中保存快照的，由于redis是用一个主线程来处理所有 client的请求，这种方式会阻塞所有client请求。所以不推荐使用。另一点需要注意的是，每次快照持久化都是将内存数据完整写入到磁盘一次，并不是增量的只同步脏数据。如果数据量大的话，而且写操作比较多，必然会引起大量的磁盘io操作，可能会严重影响性能。 

另外由于快照方式是在一定间隔时间做一次的，所以如果redis意外down掉的话，就会丢失最后一次快照后的所有修改。如果应用要求不能丢失任何修改的话，可以采用aof持久化方式。下面介绍 

Append-only file 

aof比快照方式有更好的持久化性，是由于在使用aof持久化方式时,redis会将每一个收到的写命令都通过write函数追加到文件中\(默认是appendonly.aof\)。当redis重启时会通过重新执行文件中保存的写命令来在内存中重建整个数据库的内容。当然由于os会在内核中缓存 write做的修改，所以可能不是立即写到磁盘上。这样aof方式的持久化也还是有可能会丢失部分修改。不过我们可以通过配置文件告诉redis我们想要通过fsync函数强制os写入到磁盘的时机。有三种方式如下（默认是：每秒fsync一次） appendonly yes              //启用aof持久化方式 

\# appendfsync always      //每次收到写命令就立即强制写入磁盘，最慢的，但是保证完全的持久化，不推荐使用 

appendfsynceverysec     //每秒钟强制写入磁盘一次，在性能和持久化方面做了很好的折中，推荐 

\# appendfsync no    //完全依赖os，性能最好,持久化没保证 aof的方式也同时带来了另一个问题。持久化文件会变的越来越大。例如我们调用incr test命令100次，文件中必须保存全部的100条命令，其实有99条都是多余的。因为要恢复数据库的状态其实文件中保存一条set test 100就够了。为了压缩aof的持久化文件。redis提供了bgrewriteaof命令。收到此命令redis将使用与快照类似的方式将内存中的数据以命令的方式保存到临时文件中，最后替换原来的文件。具体过程如下 1. redis调用fork ，现在有父子两个进程 

2. 子进程根据内存中的数据库快照，往临时文件中写入重建数据库状态的命令 

3.父进程继续处理client请求，除了把写命令写入到原来的aof文件中。同时把收到的写命令缓存起来。这样就能保证如果子进程重写失败的话并不会出问题。 4.当子进程把快照内容写入已命令方式写到临时文件中后，子进程发信号通知父进程。然后父进程把缓存的写命令也写入到临时文件。 

5.现在父进程可以使用临时文件替换老的aof文件，并重命名，后面收到的写命令也开始往新的aof文件中追加。 

需要注意到是重写aof文件的操作，并没有读取旧的aof文件，而是将整个内存中的数据库内容用命令的方式重写了一个新的aof文件,这点和快照有点类似。























