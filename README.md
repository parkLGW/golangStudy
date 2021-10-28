[TOC]

# Golang技术开发学习文档

## 工具类

### Syncd

Syncd是一款开源代码部署工具

#### 工作原理

* syncd通过git-ssh方式创建仓库本地副本，通过运行项目构建脚本生成可发布的代码包，使用scp串行、并行地向生产服务器分发并解包部署

* 与生产服务器通过ssh交互，因此syncd运行用户的ssh-key必须加入到生产服务器的用户ssh-key信任列表中

  ![img](https://syncd.cc/docs/assets/img/syncd-principle.png)

### Supervisor

基于python的进程管理工具，能将一个普通的命令行进程变为后台daemon，并监控进程状态，异常退出时能自动重启

安装直接通过`pip install supervisor`命令

### JumpServer跳板机



### Delve调试工具





## 技术类

### 网络相关

#### POST和PUT的区别

在HTTP中，PUT被定义为idempotent的方法，POST则不是，这是一个很重要的区别

> 如果一个方法重复执行多次，产生的效果是一样的，那就是idempotent的

#### Cookie，Session和Token

**Cookie机制**



### Mysql相关

#### **DDL/DML/DCL**

* **DDL**（Data Definition Language 数据定义语句）操作对象是表

  主要语句：Create，Drop，Alter

* **DML**（Data Manipulation Language 数据操作语句）操作对象是记录

  主要语句：Insert，Delete，Update

* **DCL** （Data Control Language 数据控制语句）操作对象是用户

  主要语句：Grant(赋予权限)，Revoke(废除权限)

#### 数据库三范式

* **第一范式（1NF）：列的原子性，即列不能够再分成其他几列**

* **第二范式（2NF）：符合1NF，表必须有一个主键，没有包含在主键中的列必须完全依赖于主键，而不能只依赖于主键的一部分**

  不符合2NF的设计会产生冗余数据

* **第三范式（3NF）：符合2NF，另外非主键列必须直接依赖于主键，不能存在传递依赖。**

#### **COLLATE**关键字

排序规则

#### [数据库备份](https://www.cnblogs.com/zhi-leaf/p/11691522.html)

使用mysqldump进行数据库备份

```sql
#导出数据库表结构
mysqldump -uroot -p --databases 数据库名 > /home/test.sql
#导出sql语句
mysqldump -uroot -h127.0.0.1 -p --databases 数据库名 --single-transaction --set-gtid-purged=off --max_allowed_packet=512M > /home/test.sql
```

```sql
# 登录数据库
mysql -uroot -p
# 执行sql脚本
source /home/test.sql
```

如果备份的库名与原库名不一样，则必须修改备份test.sql文件中的库名。如果文件很大难以用编辑器打开，可以用sed命令

```
sed -i '1,/DROP TABLE/s/`test`/`test_db`/g' test.sql
```

#### binlog三种格式

* **STATEMENT** : 每一条会修改数据的sql都会记录在binlog中

  > **优点** ：不需要记录每一行变化，减少了binlog日志量，节约了IO，提高性能
  >
  > **缺点** ：由于记录的只是执行语句，为了这些语句能在slave上正确运行，还必须记录每条语句在执行的时候的一些相关西信息。

* **ROW** : 不记录sql语句上下文相关信息，仅保存哪条记录被修改。

  > **优点** ： binlog中可以不记录执行的sql语句的上下文相关信息，仅需记录那一条记录被修改成什么样了。
  >
  > **缺点** ：所有执行的语句当记录到日志中的时候，都将以每行记录的修改来记录，可能会产生大量的日志内容

* **MIXED** : 以上两种格式的混合，一般推荐使用

#### binlog常见event

* **ROTATE_EVENT** :  当binlog文件大小达到max_binlog_size参数设置的值或执行flush logs命令时，binlog发生切换，这时会在当前使用的binlog文件末尾添加一个ROTATE_EVENT事件，将下一个binlog文件的名称和位置记录到该事件中。

* **GTID_EVENT**：事务提交时，MySQL在写binlog时会写一个GTID_EVENT，指定下一个事务的GTID，然后再写事务的binlog。主从同步时GTID_EVENT和事务的binlog都会传递到从库，从库在执行时也用同样的GTID写binlog。

* **STOP_EVENT**：当MySQL服务停止时，会在当前binlog文件尾添加一个STOP_EVENT事件表示数据库的停止

* **XID_EVENT** ：事务提交时，不论是statement还是row格式的binlog都会添加一个XID_EVENT作为事务的结束，该事件记录了该事务的ID。

* **ROWS_EVENT**：包含以下事件：`WRITE_ROWS_EVENT`，`UPDATE_ROWS_EVENT`，  `DELETE_ROWS_EVENT`

* **QUERY_EVENT**：在ROW格式下只记录`DDL`原始语句，STATEMENT格式中记录`DML`操作

  > eg：![image-20210831162123885](C:\Users\gavin\AppData\Roaming\Typora\typora-user-images\image-20210831162123885.png)

#### [GTID原理](https://www.cnblogs.com/kevingrace/p/5569753.html)

从服务器连接到主服务器之后，把自己执行过的GTID (Executed_Gtid_Set: 即已经执行的事务编码)<SQL线程> 、获取到的GTID (Retrieved_Gtid_Set: 即从库已经接收到主库的事务编号) <IO线程>发给主服务器，主服务器把从服务器缺少的GTID及对应的transactions发过去补全即可。当主服务器挂掉的时候，找出同步最成功的那台从服务器，直接把它提升为主即可。如果硬要指定某一台不是最新的从服务器提升为主， 先change到同步最成功的那台从服务器， 等把GTID全部补全了，就可以把它提升为主了。

> \- master更新数据时，会在事务前产生GTID，一同记录到binlog日志中。
> \- slave端的i/o 线程将变更的binlog，写入到本地的relay log中。
> \- sql线程从relay log中获取GTID，然后对比slave端的binlog是否有记录。
> \- 如果有记录，说明该GTID的事务已经执行，slave会忽略。
> \- 如果没有记录，slave就会从relay log中执行该GTID的事务，并记录到binlog。
> \- 在解析过程中会判断是否有主键，如果没有就用二级索引，如果没有就用全部扫描。

#### 数据库表连接

* 内连接（inner join）

  ```sql
  SELECT * FROM a INNER JOIN b ON(a.id=b.id)
  ```

  > 查询a和b的交集

  <img src="C:\Users\gavin\AppData\Roaming\Typora\typora-user-images\image-20210929110823257.png" alt="image-20210929110823257" style="zoom:100%;" />

* 左外连接（left outer join）

  ```sql
  SELECT * FROM a LEFT OUTER JOIN b ON(a.id=b.id)
  ```

  > 查询a的完全集，而b中匹配的则有值，不匹配的用null代替

  ![image-20210929110955220](C:\Users\gavin\AppData\Roaming\Typora\typora-user-images\image-20210929110955220.png)

* 右外连接（right outer join）

  ```sql
  SELECT * FROM a RIGHT OUTER JOIN b ON(a.id=b.id)
  ```

  > 查询b的完全集，a中匹配的则有值，不匹配则用null代替

  ![image-20210929111122546](C:\Users\gavin\AppData\Roaming\Typora\typora-user-images\image-20210929111122546.png)

* 全连接（union）

  ```sql
  SELECT * FROM a LEFT OUTER JOIN b ON(a.id=b.id)
  UNION
  SELECT * FROM a RIGHT OUTER JOIN b ON(a.id=b.id)
  ```

  > 左右外连接的并集，连接表包含被连接表的所有记录

  ![image-20210929111355422](C:\Users\gavin\AppData\Roaming\Typora\typora-user-images\image-20210929111355422.png)

* 交叉连接（cross join）

  ```sql
  SELECT a CROSS JOIN b WHERE a.id=b.id
  ```

  > 从连接的表中返回行的笛卡尔乘积，结果集将包含两个表中的所有行，每一行都是第一个表中的行与第二个表中的行的组合



* **Q1.**select * sort 查询时报Out of sort memory, consider increasing server sort buffer size，可能某个字段内容过大，如果不需要该字段可查询特定字段

### golang 相关

#### go mod

命令

```bash
download    download modules to local cache (下载依赖的module到本地cache))
edit        edit go.mod from tools or scripts (编辑go.mod文件)
graph       print module requirement graph (打印模块依赖图))
init        initialize new module in current directory (再当前文件夹下初始化一个新的module, 创建go.mod文件))
tidy        add missing and remove unused modules (增加丢失的module，去掉未用的module)
vendor      make vendored copy of dependencies (将依赖复制到vendor下)
verify      verify dependencies have expected content (校验依赖)
why         explain why packages or modules are needed (解释为什么需要依赖)
```



#### Gorm对象关系映射库

* Clause

#### Gin框架

* **RESTful接口**

```go
func main() {
    // Disable Console Color
    // gin.DisableConsoleColor()

    // 使用默认中间件创建一个gin路由器
    // logger and recovery (crash-free) 中间件
    router := gin.Default()

    router.GET("/someGet", getting)
    router.POST("/somePost", posting)
    router.PUT("/somePut", putting)
    router.DELETE("/someDelete", deleting)
    router.PATCH("/somePatch", patching)
    router.HEAD("/someHead", head)
    router.OPTIONS("/someOptions", options)

    // 默认启动的是 8080端口，也可以自己定义启动端口
    router.Run()
    // router.Run(":3000") for a hard coded port
}
```

* **获取请求参数**

  * GET

    ```go
    // localhost:8080/welcome?firstname=J&lastname=b
    router.Get("/welcome", func(c *gin.Context){
        firstname := c.DefaultQuery("firstname","Guest")
        lastname := c.Query("lastname")
    })
    ```

  * POST

    ```go
      router.POST("/form_post", func(c *gin.Context) {
            message := c.PostForm("message")
            nick := c.DefaultPostForm("nick", "anonymous") 
      }
    ```

    


#### cron实现定时任务

使用方法：

```go
package main
import(
    "fmt"
	cron "github.com/robfig/cron/v3"
)

func main() {
	crontab := cron.New()
	task := func() {
		fmt.Println("hello world")
	}
  // 添加定时任务, * * * * * 是 crontab,表示每分钟执行一次
	crontab.AddFunc("* * * * *", task)
  // 启动定时器
	crontab.Start()
  // 定时任务是另起协程执行的,这里使用 select 简答阻塞.实际开发中需要
  // 根据实际情况进行控制
	select {}
}
```



#### select关键字用法

select会一直阻塞直到有一个可触发的操作

select就是用来监听和channel有关的IO操作，当 IO 操作发生时，触发相应的动作。

```go
//select基本用法
select {
case <- chan1:
// 如果chan1成功读到数据，则进行该case处理语句
case chan2 <- 1:
// 如果成功向chan2写入数据，则进行该case处理语句
default:
// 如果上面都没有成功，则进入default处理流程
```

#### gRPC

* **RPC(Remote Procedure Call) ** 

  远程过程调用，就是一个节点请求另一节点提供的服务过程：

  ```go
  // Client端 
  //    Student student = Call(ServerAddr, addAge, student)
  1. 将这个调用映射为Call ID。
  2. 将Call ID，student（params）序列化，以二进制形式打包
  3. 把2中得到的数据包发送给ServerAddr，这需要使用网络传输层
  4. 等待服务器返回结果
  5. 如果服务器调用成功，那么就将结果反序列化，并赋给student，年龄更新
  
  // Server端
  1. 在本地维护一个Call ID到函数指针的映射call_id_map，可以用Map<String, Method> callIdMap
  2. 等待客户端请求
  3. 得到一个请求后，将其数据包反序列化，得到Call ID
  4. 通过在callIdMap中查找，得到相应的函数指针
  5. 将student（params）反序列化后，在本地调用addAge()函数，得到结果
  6. 将student结果序列化后通过网络返回给Client
  ```

  **对象注册为RPC服务规则** ：**方法只能有两个可序列化的参数，其中第二个参数是指针类型，并返回一个error类型，同时必须是公开的方法**

  `rpc.Register`函数调用会将对象类型中所有满足RPC规则的对象方法注册为RPC函数

  **RPC服务接口规范**

  * 服务名字
  * 服务要实现的详细方法列表
  * 这侧该类型服务的函数
  

#### WaitGroup并发控制

sync.WaitGroup可以解决同步阻塞等待的问题

`WaitGroup` 对象内部有一个计数器，最初从0开始，它有三个方法：`Add(), Done(), Wait()` 用来控制计数器的数量。`Add(n)` 把计数器加`n` ，`Done()` 每次把计数器`-1` ，`wait()` 会阻塞代码的运行，直到计数器地值减为`0`。

**数据结构：**

```go
type WaitGroup struct {
	noCopy noCopy
	state1 [12]byte
	sema   uint32
}
```

#### pprof性能分析

[使用方法](https://www.jianshu.com/p/a054fda87918)：

```go
package main

import (
    // 略
    _ "net/http/pprof" // 会自动注册 handler 到 http server，方便通过 http 接口获取程序运行采样报告
    // 略
)

func main() {
    // 略

    runtime.GOMAXPROCS(1) // 限制 CPU 使用数，避免过载
    runtime.SetMutexProfileFraction(1) // 开启对锁调用的跟踪
    runtime.SetBlockProfileRate(1) // 开启对阻塞操作的跟踪

    go func() {
        // 启动一个 http server，注意 pprof 相关的 handler 已经自动注册过了
        if err := http.ListenAndServe(":6060", nil); err != nil {
            log.Fatal(err)
        }
        os.Exit(0)
    }()

    // 略
}
```

保持程序运行，然后在浏览器访问http://localhost:6060/debug/pprof

![image-20210727103102762](C:\Users\gavin\AppData\Roaming\Typora\typora-user-images\image-20210727103102762.png)

![image-20210827135018589](C:\Users\gavin\AppData\Roaming\Typora\typora-user-images\image-20210827135018589.png)

通过在终端输入命令`go tool pprof http://localhost:6060/debug/pprof/heap ` 

**输入top 显示内存占用情况**：

![image-20210727110731838](C:\Users\gavin\AppData\Roaming\Typora\typora-user-images\image-20210727110731838.png)

**输入list 函数查看详细占用信息**：

![image-20210727110850161](C:\Users\gavin\AppData\Roaming\Typora\typora-user-images\image-20210727110850161.png)

**输入web 查看图形化显示**

![image-20210727111034945](C:\Users\gavin\AppData\Roaming\Typora\typora-user-images\image-20210727111034945.png)

#### [Context上下文机制](https://www.cnblogs.com/zhangboyu/p/7456606.html)

上下文一般理解为程序单元（goroutine）的一个运行状态、现场、快照。每个Goroutine在执行之前，都要先知道程序当前的执行状态，通常将这些执行状态封装在一个`Context`变量中，传递给要执行的Goroutine中。

**其定义如下**：

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

- `Deadline`会返回一个超时时间，Goroutine获得了超时时间后，例如可以对某些io操作设定超时时间。
- `Done`方法返回一个信道（channel），当`Context`被撤销或过期时，该信道是关闭的，即它是一个表示Context是否已关闭的信号。
- 当`Done`信道关闭后，`Err`方法表明`Contex`t被撤的原因。
- `Value`可以让Goroutine共享一些数据，当然获得数据是协程安全的。但使用这些数据的时候要注意同步，比如返回了一个map，而这个map的读写则要加锁。

`Context`接口没有提供方法来设置其值和过期时间，也没有提供方法直接将其自身撤销。也就是说，`Context`不能改变和撤销其自身。

Context结构像一棵树，叶子节点须总是由根节点衍生出来的。要创建Context树，第一步就是要得到根节点，`context.Background`函数的返回值就是根节点：

```go
func Background() Context
```

该函数返回空的Context，该Context一般由接收请求的第一个Goroutine创建，是与进入请求对应的Context根节点，它不能被取消、没有值、也没有过期时间。它常常作为处理Request的顶层context存在。

**创建子节点的函数：**

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key interface{}, val interface{}) Context
```

函数都接收一个`Context`类型的参数`parent`，并返回一个`Context`类型的值，这样就层层创建出不同的节点。子节点是从复制父节点得到的，并且根据接收参数设定子节点的一些状态值，接着就可以将子节点传递给下层的Goroutine了

### 时间轮算法



### 分布式锁

#### 分布式锁是什么

在分布式集群系统中，为保证一个方法或属性在高并发情况下的同一时间只能被同一个线程执行，需要一种跨机器互斥机制来控制共享资源的访问。

#### 分布式锁应具备的条件

1. 在分布式系统环境下，一个方法在同一时间只能被一个机器的一个线程执行
2. **高可用**的获取锁与释放锁
3. **高性能**的获取锁与释放锁
4. 具备**可重入**特性
5. 具备**锁失效机制**，防止死锁
6. 具备**非阻塞锁**特性，即没有获取到锁将直接返回获取锁失败

#### 分布式锁三种实现方式

> **[强一致性、弱一致性和最终一致性](https://zhuanlan.zhihu.com/p/67949045)**
>
> **强一致性**：即复制是同步的，可以保证从库有与主库一致的数据
>
> **弱一致性**：即复制是异步的
>
> **最终一致性**：用户从异步从库读取时，如果此异步从库落后，可能会看到过时的信息，但等待一段时间，从库最终会赶上并与主库保持一致，即最终一致性

1. ##### 基于数据库实现分布式锁

   在数据库中创建一个表，表中包含方法名等字段，并在方法名字段上创建唯一索引，要执行某个方法，就使用这个方法名向表中插入数据，成功则获取锁，执行完成后删除对应的行数据释放锁

   (1) 创建一个表

   ```sql
   DROP TABLE IF EXISTS `method_lock`;
   CREATE TABLE `method_lock` (
     `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
     `method_name` varchar(64) NOT NULL COMMENT '锁定的方法名',
     `desc` varchar(255) NOT NULL COMMENT '备注信息',
     `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
     PRIMARY KEY (`id`),
     UNIQUE KEY `uidx_method_name` (`method_name`) USING BTREE
   ) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8 COMMENT='锁定中的方法';
   ```

   （2）要执行某个方法，就使用这个方法名向表中插入数据

   ```sql
   INSERT INTO method_lock (method_name, desc) VALUES ('methodName', '测试的methodName');
   ```

   由于对method_name做了唯一性约束，如果有多个请求同时提交到数据库，数据库会保证只有一个操作可以成功。

   （3）成功插入则获取锁，执行完成后删除对应数据行释放锁

   ```sql
   delete from method_lock where method_name ='methodName';
   ```

   **缺点**:

   * 受数据库可用性和性能的影响
   * 不具备可重入性，因为同一个线程在释放锁之前，行数据一直存在，无法再次成功插入数据，所以，需要在表中新增一列，用于记录当前获取到锁的机器和线程信息，在再次获取锁的时候，先查询表中机器和线程信息是否和当前机器和线程相同，若相同则直接获取锁
   * 没有锁失效机制，因为有可能出现成功插入数据后，服务器宕机了，对应的数据没有被删除，当服务恢复后一直获取不到锁，所以，需要在表中新增一列，用于记录失效时间，并且需要有定时任务清除这些失效的数据；
   * 不具备阻塞锁特性，获取不到锁直接返回失败，所以需要优化获取逻辑，循环多次去获取
   * 依赖数据库需要一定的资源开销

   

2. ##### 基于缓存（Redis等）实现分布式锁

   > 用到的Redis命令介绍
   >
   > (1) **setnx** 
   >
   > setnx key val:当且仅当key不存在时，set一个key为val的字符串，返回1；若key存在，则只返回0
   >
   > (2) **expire**
   >
   > expire key timeout:为key设置一个超时时间，单位为秒，超过时间自动释放锁
   >
   > (3) **delete**
   >
   > delete key:删除key

   * 获取锁的时候，使用setnx加锁，并使用expire为锁加一个超时时间，超时则自动释放锁，锁的value值为一个随机生成的UUID，通过此在释放锁的时候进行判断。
   * 获取锁的时候还设置一个获取的超时时间，若超过这个时间则放弃获取锁
   * 释放锁的时候，通过UUID判断是不是该锁，若是该锁，则执行delete释放锁

   ```python
   #连接redis
   redis_client = redis.Redis(host="localhost",
                              port=6379,
                              password=password,
                              db=10)
   
   #获取一个锁
   lock_name：锁定名称
   acquire_time: 客户端等待获取锁的时间
   time_out: 锁的超时时间
   def acquire_lock(lock_name, acquire_time=10, time_out=10):
       """获取一个分布式锁"""
       identifier = str(uuid.uuid4())
       end = time.time() + acquire_time
       lock = "string:lock:" + lock_name
       while time.time() < end:
           if redis_client.setnx(lock, identifier):
               # 给锁设置超时时间, 防止进程崩溃导致其他进程无法获取锁
               redis_client.expire(lock, time_out)
               return identifier
           elif not redis_client.ttl(lock):
               redis_client.expire(lock, time_out)
           time.sleep(0.001)
       return False
   
   #释放一个锁
   def release_lock(lock_name, identifier):
       """通用的锁释放函数"""
       lock = "string:lock:" + lock_name
       pip = redis_client.pipeline(True)
       while True:
           try:
               pip.watch(lock)
               lock_value = redis_client.get(lock)
               if not lock_value:
                   return True
   
               if lock_value.decode() == identifier:
                   pip.multi()
                   pip.delete(lock)
                   pip.execute()
                   return True
               pip.unwatch()
               break
           except redis.excetions.WacthcError:
               pass
       return False
   ```

   

3. ##### 基于zookeeper实现分布式锁

zookeeper是一个为分布式应用提供一致性服务的开源组件，它内部是一个分层的文件系统目录树结构，规定同一个目录下只能有一个唯一文件名。基于zookeeper实现分布式锁：

(1) 创建一个目录mylock

(2) 线程A想获取锁就在mylock下创建临时顺序节点

(3) 获取mylock下的所有子节点，然后获取比自己小的兄弟节点，如果不存在，则说明当前线程顺序号最小，获得锁

(4) 线程B获取所有节点，判断自己不是最小节点，设置监听比自己次小的节点

(5) 线程A处理完，删除自己的节点，线程B监听到变更事件，判断自己是不是最小节点，如果是则获得锁

**优点**：具备高可用、可重用、阻塞锁特性，可解决失效死锁问题

**缺点**：因为需要频繁的创建和删除节点，性能上不如Redis方式



### Kafka

#### Kafka是什么

**kafka是一个分布式的、分区化、可复制提交的日志服务**

**特点**：

* Kafka是分布式的，其所有的构件borker(服务端集群)、producer(消息生产)、consumer(消息消费者)都可以是分布式的。
* 消息生产时可以使用一个标识topic来区分，且可以进行分区；每个分区都是一个顺序的、不可变的消息队列，并且可持续增加。
* 同时为发布和订阅提供高吞吐量。
* 消息被处理的状态在consumer端维护，而不是由server端维护，失败时能自动平衡。

**应用场景**

* 监控
* 消息队列
* 站点用户活动追踪
* 流处理
* 日志聚合
* 持久性日志

**基础概念**

```
  1.Topic(话题)：Kafka中用于区分不同类别信息的类别名称。由producer指定
    2.Producer(生产者)：将消息发布到Kafka特定的Topic的对象(过程)
    3.Consumers(消费者)：订阅并处理特定的Topic中的消息的对象(过程)
    4.Broker(Kafka服务集群)：已发布的消息保存在一组服务器中，称之为Kafka集群。集群中的每一个服务器都是一个代理(Broker). 消费者可以订阅一个或多个话题，并从Broker拉数据，从而消费这些已发布的消息。
    5.Partition(分区)：Topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列。partition中的每条消息都会被分配一个有序的id（offset）
    Message：消息，是通信的基本单位，每个producer可以向一个topic（主题）发布一些消息。
```

**架构介绍**

![image-20210612143051457](C:\Users\gavin\AppData\Roaming\Typora\typora-user-images\image-20210612143051457.png)



**工作流程**

![](C:\Users\gavin\AppData\Roaming\Typora\typora-user-images\image-20210612142636317.png)

```
    1.⽣产者从Kafka集群获取分区leader信息
    2.⽣产者将消息发送给leader
    3.leader将消息写入本地磁盘
    4.follower从leader拉取消息数据
    5.follower将消息写入本地磁盘后向leader发送ACK
    6.leader收到所有的follower的ACK之后向生产者发送ACK
```

**选择partition的原则**

```
  1.partition在写入的时候可以指定需要写入的partition，如果有指定，则写入对应的partition。
  2.如果没有指定partition，但是设置了数据的key，则会根据key的值hash出一个partition。
  3.如果既没指定partition，又没有设置key，则会采用轮询⽅式，即每次取一小段时间的数据写入某
    个partition，下一小段的时间写入下一个partition
```



### ElasticSearch

[参考](https://www.cnblogs.com/ljhdo/p/4486978.html)

#### Endpoint

* **_search**：用于搜索查询，返回查询结果
* **_query**：只用于将查询的结果删除`DELETE /_query?q=user:park`
* **_analyze**：用于对查询参数进行分析，并返回分析结果
* **_count**：获取满足查询条件的文档数量`GET /_count?q=user:park`
* **_explain**：验证指定文档是否满足查询条件`GET index/type/1/_explain?q=message:search`
* **_exists**：检查是否有文档满足查询条件`GET /_search/exists?q=user:kimchy`

#### 倒排索引

倒排索引是实现“单词-文档矩阵”的一种具体存储形式，通过倒排索引，可以根据单词快速获取包含这个单词的文档列表。倒排索引主要由两个部分组成：“单词词典”和“倒排文件”。

> **单词词典(Lexicon)**：搜索引擎的通常索引单位是单词，单词词典是由文档集合中出现过的所有单词构成的字符串集合，单词词典内每条索引项记载单词本身的一些信息以及指向“倒排列表”的指针。
>
> **倒排列表(PostingList)**：倒排列表记载了出现过某个单词的所有文档的文档列表及单词在该文档中出现的位置信息，每条记录称为一个倒排项(Posting)。根据倒排列表，即可获知哪些文档包含某个单词。

#### Suggest 自动补全

suggester基本运作原理：将输入的文本分解为token，然后再索引的字典中查找相似的term并且返回。根据使用场景不同，elasticsearch中涉及了4中类别的suggester:

* Term Suggester

  提供基于单个词项的拼写纠错方法

* Phrase Suggester

  可返回完整短语建议而不是单个词项建议

* Completion Suggester

  针对自动补全场景，只能用于前缀查找

* Context Suggester

#### [分词器](https://www.cnblogs.com/qdhxhz/p/11585639.html)

ik分词器

#### 搜索相关性

决定文档得分的三个主要因素：

* 字词频率（TF）：搜索关键词在正在搜索的文档中的字段中出现的次数越多，则该文档就越相关
* 反向文档频率（IDF）：要搜索的字段中包含搜索关键词的文档越多，则该词的重要性就越低
* 字段长度：文档在非常短的字段中包含搜索词，则比文档在较长的字段中包含搜索词更相关。

bool查询提供了4个语句，must/filter/should/must_not，其中filter/must_not属于过滤器，must/should属于查询器

* 过滤器不计算相关性评分，因为被过滤掉的内容不会影响返回内容的排序
* 过滤器可以使用ES内部的缓存，所以过滤器可以提高查询速度

**对ES Query DSL的优化**

1. multi_match

   会把多个字段的匹配转换成多个match查询组合，挨个对字段进行match查询。match 执行查询时，先把查询关键词经过 search_analyzer 设置的分析器分析，再把分析器得到的结果挨个放进 bool 查询中的 should 语句，这些 should 没有权重与顺序的差别，并且只要命中一个should 语句的文档都会被返回


   

2. bool查询的filter

3. match_phrase

   要求必须命中所有分词，并且返回的文档命中的词也要按照查询短语的顺序，词的间距可以使用slop设置。slop参数告诉match_phrase查询词条相隔多远时仍能将文档视为匹配

   ![image-20211012111622701](C:\Users\gavin\AppData\Roaming\Typora\typora-user-images\image-20211012111622701.png)

4. boost调整权重

5. function_score增加评分因素（如时间、热度、推荐权重等）

   * script_score
   * weight
   * random_score
   * field_value_factor：使用某字段影响总分数（如点赞 评论数）
   * decay_function：gauss、exp、linear





#### 隐马尔可夫模型HMM

[参考资料](https://cloud.tencent.com/developer/article/1831564)

HMM的典型模型是一个五元组：

* StatusSet：状态值集合。(B:begin,M:middle,E:end,S:single)
* ObservedSet：观察值集合。句子中所有字的集合
* TransProbMatrix：转移概率模型。根据有限历史性假设得到，是一个4x4的矩阵（即BMESxBMES）数值是概率求对数后的值
* EmitProbMatrix：发射概率模型。根据观察值独立性假设，是位置到单字的发射概率，P(“和”|M)表示"和"字出现在某个词中间的概率
* InitStatus：初始状态分布。句子的第一个字属于B,M,E,S这四种状态的概率

HMM模型的三个基本假设：

* 有限历史性假设（当前状态只与上一个状态相关）

  P(status<sub>i</sub>|status<sub>i-1</sub>,status<sub>i-2</sub>,...,status<sub>1</sub>)=P(Status<sub>i</sub>|status<sub>i-1</sub>)

* 齐次性假设（状态和当前时刻无关）

  P(status<sub>i</sub>|status<sub>i-1</sub>)=P(Status<sub>j</sub>|status<sub>j-1</sub>)

* 观察值独立性假设（观察值只取决于当前状态值）

  P(Observed<sub>i</sub>|status<sub>i</sub>,status<sub>i-1</sub>,status<sub>i-2</sub>,...,status<sub>1</sub>)=P(Observed<sub>i</sub>|status<sub>i</sub>)

**viterbi算法：**

```c++
初始状态分布InitStatus
    
B:-0.26268660809250016
E:-3.14e+100
M:-3.14e+100
S:-1.4652633398537678

且由EmitProbMatrix可以得出

Status(B) -> Observed(小)  :  -5.79545
Status(E) -> Observed(小)  :  -7.36797
Status(M) -> Observed(小)  :  -5.09518
Status(S) -> Observed(小)  :  -6.2475

所以可以初始化 weight[i][0] 的值如下：

weight[0][0] = -0.26268660809250016 + -5.79545 = -6.05814
weight[1][0] = -3.14e+100 + -7.36797 = -3.14e+100
weight[2][0] = -3.14e+100 + -5.09518 = -3.14e+100
weight[3][0] = -1.4652633398537678 + -6.2475 = -7.71276

注意上式计算的时候是相加而不是相乘，因为之前取过对数的原因。

//遍历句子，下标i从1开始是因为刚才初始化的时候已经对0初始化结束了
for(size_t i = 1; i < 15; i++)
{
    // 遍历可能的状态
    for(size_t j = 0; j < 4; j++) 
    {
        weight[j][i] = MIN_DOUBLE;
        path[j][i] = -1;
        //遍历前一个字可能的状态
        for(size_t k = 0; k < 4; k++)
        {
            double tmp = weight[k][i-1] + _transProb[k][j] + _emitProb[j][sentence[i]];
            if(tmp > weight[j][i]) // 找出最大的weight[j][i]值
            {
                weight[j][i] = tmp;
                path[j][i] = k;
            }
        }
    }
}
```















### 敏感词过滤

#### Trie树



#### AC自动机





























