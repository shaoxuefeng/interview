# Redis
内存型数据库，高性能的key-value数据库。

<br/>

### 基本数据类型
**1.字符串:** 简单动态字符串(simple dynamic string, SDS)的抽象类型  
**2. 字符串列表:**  压缩列表&链表  
&emsp;&emsp; 当每个元素小于(64字节)且总元素数小于512时，选择压缩列表  
**3. 集合**  intset(整数集合) & hashtable(哈希表)  
&emsp;&emsp; 当集合数据量小于512时，选用intset  
**4. 有序集合**  ziplist (压缩列表)&skiplist (跳跃表)  
&emsp;&emsp;  当每个元素小于(64字节)且总元素数小于128时，选择压缩列表   
**5. 字典**  散列表    

参考:[https://blog.laoyu.site/2019/redis/Redis对象——Redis对象系统简介/](https://blog.laoyu.site/2019/redis/Redis对象——Redis对象系统简介/)

<br/>

#### redis为什么快
1.  完全基于内存，几乎没有IO
2.  数据结构简单，数据的操作也简单，基本事件复杂度都是O(1)
3.  单线程，没有上下文切换和CPU切换，也不需要锁
4.  底层IO多路复用/epoll

**单线程原因:**  
&emsp;&emsp;  因为Redis是基于内存的操作，CPU不是Redis的瓶颈，Redis的瓶颈最有可能是机器内存的大小或者网络带宽。

<br/>

#### redis持久化
**RDB(Redis Database):** 制定时间间隔，将redis存储的数据生成快照，写入磁盘。  
&emsp;&emsp;  优点: 整个Redis数据库将只包含一个文件，这对于文件备份而言是非常完美的。  
&emsp;&emsp;  缺点: RDB方式实现持久化，一旦Redis异常退出，就会丢失最后一次快照以后更改的所有数据。  
**AOF(Append Only File):** 以日志形式记录服务器每一个操作，在Redis服务器启动之初会读取该文件来重新构建数据库，以保证启动后数据库中的数据是完整的。     
&emsp;&emsp;  优点: 可以保证每次修改都记录到日志文件中，保证数据不丢失。  
&emsp;&emsp;  缺点: AOF生成的文件，远远大于RDB快照；恢复速度也慢于RDB快照。  
**备注:** Redis允许同时开启AOF和RDB，既保证了数据安全又使得进行备份等操作十分容易。同时采用，在重启恢复时，采用的是AOF方式恢复，减少文件丢失。  

<br/>

#### redis主从同步
1. master 执行sync命令，生成RDB， 此时的执行的redis操作，都存到redis的缓存中
2. 发送RDB文件到从服务器，
3. 从服务器保存RDB文件到磁盘，并且读RDB文件到内存中，
4. 数据同步期间的缓存，将以redis协议的方式，同步给从服务器。