# Connection失效的校验-TestOnBorrow等参数
[Apache Tomcat 8--The Tomcat JDBC Connection Pool](https://tomcat.apache.org/tomcat-8.0-doc/jdbc-pool.html)

1. testWhileIdle
2. testOnBorrow
3. testOnReturn
4. timeBetweenEvictionRunsMillis
5. ValidationInterval
6. numTestsPerEvictionRun ： Property not used in tomcat-jdbc-pool.

druid和tomcat连接池，对testXXXX属性默认都设置为了false（并不完全是），这可能导致`getConnections()`时，得到一个不可用的连接，从而导致报错：
```
The last packet successfully received from the server was 19,956 milliseconds ago.  The last packet sent successfully to the server was 32 milliseconds ago.
    at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
    at com.mysql.jdbc.StatementImpl.executeQuery(StatementImpl.java:1571)
Caused by: java.io.EOFException: Can not read response from server. Expected to read 4 bytes, read 0 bytes before connection was unexpectedly lost.
    at com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:3143)
    at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:3597)
    ... 8 more
```
之所以考虑将其默认值设置为false，主要是从性能上进行考虑的。

### tomcat源码对该属性的描述
tomcat-jdbc:8.5.5

#### testOnBorrow
默认值为false，主要是考虑到性能问题，可以尝试配置`validationInterval`来提高性能。（In order to have a more efficient validation, see `validationInterval`）

Connection在从Connection-Pools拿出之前，是否对Connection其进行校验
```
/**
 * The indication of whether objects will be validated before being borrowed from the pool.
 * If the object fails to validate, it will be dropped from the pool, and we will attempt to borrow another.
 * NOTE - for a true value to have any effect, the validationQuery parameter must be set to a non-null string.
 * Default value is false
 * In order to have a more efficient validation, see {@link #setValidationInterval(long)}
 * @param testOnBorrow set to true if validation should take place before a connection is handed out to the application
 * @see #getValidationInterval()
 */
public void setTestOnBorrow(boolean testOnBorrow);
```

但是在 spring boot 集成mybatis时候 却默认修改了配置:    
![](http://dl2.iteye.com/upload/attachment/0122/8647/38f2fcc5-854b-32ae-98b7-6a692aa9896a.png)    

1. 性能问题。

目前网站的应用大部分的瓶颈还是在I/O这一块，大部分的I/O还是在数据库的这一层面上，每一个请求可能会调用10来次SQL查询，如果不走事务，一个请求会重复获取链接，如果每次获取链接,比如在testOnBorrow都进行validateObject，性能开销不是很能接受，可以假定一次SQL操作消毫0.5~1ms(一般走了网络请求基本就这数)

#### testOnReturn
Connection被返回到Conn-pools时是否对Connection进行校验。感觉用处不大
```
 /**
  * The indication of whether objects will be validated after being returned to the pool.
  * If the object fails to validate, it will be dropped from the pool.
  * NOTE - for a true value to have any effect, the validationQuery parameter must be set to a non-null string.
  * Default value is false
  * In order to have a more efficient validation, see {@link #setValidationInterval(long)}
  * @return true if validation should take place after a connection is returned to the pool
  * @see #getValidationInterval()
  */
 public boolean isTestOnReturn();
```

#### testWhileIdle（重点关注）
在Connection空闲时，对conn-pools中的Connection进行校验。   
一般需要配合`timeBetweenEvictionRunsMillis`做优化

```
 /**
  * Set to true if query validation should take place while the connection is idle.
  * @param testWhileIdle true if validation should take place during idle checks
  * @see #setTimeBetweenEvictionRunsMillis(int)
  */
 public void setTestWhileIdle(boolean testWhileIdle);

```

#### timeBetweenEvictionRunsMillis
空闲Connection的校验，清理废弃的Connection，resize空闲Connection的大小。校验时间间隔，默认5s
```
 /**
  * The number of milliseconds to sleep between runs of the idle connection validation, abandoned cleaner
  * and idle pool resizing. This value should not be set under 1 second.
  * It dictates how often we check for idle, abandoned connections, and how often we validate idle connection and resize the idle pool.
  * The default value is 5000 (5 seconds)
  * @param timeBetweenEvictionRunsMillis the sleep time in between validations in milliseconds
  */
 public void setTimeBetweenEvictionRunsMillis(int timeBetweenEvictionRunsMillis);
```

#### ValidationInterval
避免过度校验，设置校验频率。   
如果一个Connection需要进行校验，但是在ValidationInterval毫秒之内已经校验过了，则这次不会对该Connection进行重复校验。默认值3s。
```
 /**
 * avoid excess validation, only run validation at most at this frequency - time in milliseconds.
 * If a connection is due for validation, but has been validated previously
 * within this interval, it will not be validated again.
 * The default value is 30000 (30 seconds).
 * @param validationInterval the validation interval in milliseconds
 */
public void setValidationInterval(long validationInterval);
```

#### 总结
从上面的描述来看，如果`testOnBorrow`和`testReturn`设置为true,可能会导致频繁的Connection校验，很耗性能。可以通过参数`ValidationInterval`、`timeBetweenEvictionRunsMillis`、`testWhileIdle`的配合，来改善性能。



---
# DBCP连接池TestOnBorrow的坑
生产环境连接池TestOnBorrow设置为false，导致有时获取的连接不可用。   
含义：在使用时测试Connection是否可用。

该问题的产生，和tcp的`close_wait`状态有关，可以查询`close_wait`的产生原因去理解。![close_wait状态](http://dl2.iteye.com/upload/attachment/0122/8582/9201bbf1-075c-360a-a2f5-bc0881b07f1f.gif)    

## TestOnBorrow=false
设为false时，在使用时不会测试pools中的Connection是否可用，因此可能获取到不可用的Connection。     
TestOnBorrow=false时，由于不检测池里连接的可用性，于是假如**连接池中的连接被数据库关闭**了，应用通过连接池getConnection时，都可能获取到这些不可用的连接，且这些连接如果不被其他线程回收的话，它们不会被连接池废除，也不会重新被创建，占用了连接池的名额。

项目本身作为server，数据库Connections被关闭，客户端调用服务端就会出现大量的timeout，客户端设置了超时时间，然而主动断开，服务端必然出现close_wait。


```
The last packet successfully received from the server was 19,956 milliseconds ago.  The last packet sent successfully to the server was 32 milliseconds ago.
    at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
    at com.mysql.jdbc.StatementImpl.executeQuery(StatementImpl.java:1571)
Caused by: java.io.EOFException: Can not read response from server. Expected to read 4 bytes, read 0 bytes before connection was unexpectedly lost.
    at com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:3143)
    at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:3597)
    ... 8 more
```

## 当TestOnBorrow=true时，有两种情况
#### 情况1
集群某实例宕掉时，如果连接刚好不处于通信阶段，tcp连接正处于CLOSE_WAIT状态或已关闭，当应用通过连接池getConnection时，在borrow时会检测连接，由于连接已关闭，于是报了如下报错，并重新建立新连接，此时的新连接到集群的其他实例上了。后面能正常通信。
```
com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure

The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.
    at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
Caused by: java.io.EOFException: Can not read response from server. Expected to read 4 bytes, read 0 bytes before connection was unexpectedly lost.
    at com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:3143)
    at com.mysql.jdbc.MysqlIO.readPacket(MysqlIO.java:597)
    ... 21 more
```

#### 情况2
集群某实例宕掉时，如果连接刚好处于通信阶段，由于客户端无法立即感知服务端已断连接，它可能会报如下错误，等待服务端的响应超时报错。当应用通过连接池getConnection时，在borrow时会检测连接，由于连接已关闭，于是报了如下报错，并重新建立新连接，此时的新连接到集群的其他实例上了。后面能正常通信。
```
com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure

The last packet successfully received from the server was 10,538 milliseconds ago.  The last packet sent successfully to the server was 10,306 milliseconds ago.
    at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
Caused by: com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure
```


---

## druid的推荐配置：
```
<property name="validationQuery" value="SELECT 'x'" />
<property name="testWhileIdle" value="true" />
<property name="testOnBorrow" value="false" />
<property name="testOnReturn" value="false" />
```
1. testOnBorrow和testOnReturn在生产环境一般是不开启的，主要是性能考虑。失效连接主要通过testWhileIdle保证，如果获取到了不可用的数据库连接，一般由应用处理异常。

2. 对于常规的数据库连接池，testOnBorrow等配置参数的含义和最佳实践可以参考官方文档。

3. 数据源库连接池的实现原理与dropwizard无关，既然mysql server的wait_timeout等参数被设置为30秒，那么就会主动关闭不活跃的客户端连接，**几个test参数设置为true可以通过充分的检测移除不可用连接，并重新创建新的连接，保证应用都获取到健康的连接**。

4. my.conf中的**wait_timeout**参数和**interactive_timeout参数**默认是28800秒，也就是8小时。**一般在生产环境这个数值会被设置为7天甚至30天，目的是保证mysql不会因为流量稀少而主动关闭session**. 至于是否会导致大量的sleep连接，这个请在理解以上原理后，自行思考吧

---

## 常见问题
#### 问题例一：
MySQL8小时问题，Mysql服务器默认连接的“wait_timeout”是8小时，也就是说一个connection空闲超过8个小时，Mysql将自动断开该 connection。

但是DBCP连接池并不知道连接已经断开了，如果程序正巧使用到这个已经断开的连接，程序就会报错误。

#### 问题例二：
以前还使用Sybase数据库，由于某种原因，数据库死了后重启、或断网后恢复。
等了约10分钟后，DBCP连接池中的连接还都是不能使用的（断开的），访问数据应用一直
报错，最后只能重启Tomcat问题才解决 。

#### 解决方案：
- 方案1、定时对连接做测试，测试失败就关闭连接。
- 方案2、控制连接的空闲时间达到N分钟，就关闭连接，（然后可再新建连接）。

以上两个方案使用任意一个就可以解决以述两类问题。如果只使用方案2，建议 N <= 5分钟。连接断开后最多5分钟后可恢复。

也可混合使用两个方案，建议 N = 30分钟。

下面就是DBCP连接池，同时使用了以上两个方案的配置配置
- `validationQuery = "SELECT 1"`：验证连接是否可用，使用的SQL语句
- `testWhileIdle = "true"`：指明连接是否被空闲连接回收器(如果有)进行检验.如果检测失败,则连接将被从池中去除.
- `testOnBorrow = "false"`：借出连接时不要测试，否则很影响性能
- `timeBetweenEvictionRunsMillis = "30000"`：每30秒运行一次空闲连接回收器
- `minEvictableIdleTimeMillis = "1800000"`：  池中的连接空闲30分钟后被回收,默认值就是30分钟。
- `numTestsPerEvictionRun="3"`： 在每次空闲连接回收器线程(如果有)运行时检查的连接数量，默认值就是3.

#### 解释：
配置`timeBetweenEvictionRunsMillis = "30000"`后，每30秒运行一次空闲连接回收器（独立线程）。并每次检查3个连接，如果连接空闲时间超过30分钟就销毁。销毁连接后，连接数量就少了，如果小于`minIdle`数量，就新建连接，维护数量不少于minIdle，过行了新老更替。

`testWhileIdle = "true"` 表示每30秒，取出3条连接，使用validationQuery = "SELECT 1" 中的SQL进行测试 ，测试不成功就销毁连接。销毁连接后，连接数量就少了，如果小于minIdle数量，就新建连接。

`testOnBorrow = "false"` 一定要配置，因为它的默认值是true。false表示每次从连接池中取出连接时，不需要执行validationQuery = "SELECT 1" 中的SQL进行测试。若配置为true,对性能有非常大的影响，性能会下降7-10倍。所在一定要配置为false.

每30秒，取出numTestsPerEvictionRun条连接（本例是3，也是默认值），发出"SELECT 1" SQL语句进行测试 ，测试过的连接不算是“被使用”了，还算是空闲的。连接空闲30分钟后会被销毁。

## DBCP连接池配置参数注意事项
#### maxIdle值与maxActive值应配置的接近。
因为，当连接数超过maxIdle值后，刚刚使用完的连接（刚刚空闲下来）会立即被销毁。而不是我想要的空闲M秒后再销毁起一个缓冲作用。这一点DBCP做的可能与你想像的不一样。

若maxIdle与maxActive相差较大，在高负载的系统中会导致频繁的创建、销毁连接，连接数在maxIdle与maxActive间快速频繁波动，这不是我想要的。

高负载系统的maxIdle值可以设置为与maxActive相同或设置为-1(-1表示不限制)，让连接数量在minIdle与maxIdle间缓冲慢速波动。

#### timeBetweenEvictionRunsMillis建议设置值
`initialSize="5"`，会在tomcat一启动时，创建5条连接，效果很理想。

但同时我们还配置了`minIdle="10"`，也就是说，最少要保持10条连接，那现在只有5条连接，哪什么时候再创建少的5条连接呢？
1、等业务压力上来了， DBCP就会创建新的连接。
2、配置`timeBetweenEvictionRunsMillis=“时间”`,DBCP会启用独立的工作线程定时检查，补上少的5条连接。销毁多余的连接也是同理。


## 连接销毁的逻辑
DBCP的连接数会在:`0 - minIdle - maxIdle - maxActive`之间变化。变化的逻辑描述如下：

默认未配置initialSize(默认值是0)和timeBetweenEvictionRunsMillis参数时，刚启动tomcat时，连接数是0。当应用有一个并发访问数据库时DBCP创建一个连接。

目前连接数量还未达到minIdle，但DBCP也不自动创建新连接已使数量达到minIdle数量（没有一个独立的工作线程来检查和创建）。

随着应用并发访问数据库的增多，连接数也增多，但都与minIdle值无关，很快minIdle被超越，minIdle值一点用都没有。

直到连接的数量达到maxIdle值，这时的连接都是只增不减的。 再继续发展，连接数再增多并超过maxIdle时，使用完的连接（刚刚空闲下来的）会立即关闭，总体连接的数量稳定在maxIdle但不会超过maxIdle。

但活动连接（在使用中的连接）可能数量上瞬间超过maxIdle，但永远不会超过maxActive。

这时如果应用业务压力小了，访问数据库的并发少了，连接数也不会减少（没有一个独立的线程来检查和销毁），将保持在maxIdle的数量。

默认未配置initialSize(默认值是0)，但配置了timeBetweenEvictionRunsMillis=“30000”（30秒）参数时，刚启动tomcat时，连接数是0。马上应用有一个并发访问数据库时DBCP创建一个连接。

目前连接数量还未达到minIdle，每30秒DBCP的工作线程检查连接数是否少于minIdle数量，若少于就创建新连接直到达到minIdle数量。

随着应用并发访问数据库的增多，连接数也增多，直到达到maxIdle值。这期间每30秒DBCP的工作线程检查连接是否空闲了30分钟，若是就销毁。但此时是业务的高峰期，是不会有长达30分钟的空闲连接的，工作线程查了也是白查，但它在工作。到这里连接数量一直是呈现增长的趋势。

**当连接数再增多超过maxIdle时，使用完的连接(刚刚空闲下来)会立即关闭，总体连接的数量稳定在maxIdle**。停止了增长的趋势。但活动连接（在使用中的连接）可能数量上瞬间超过maxIdle，但永远不会超过maxActive。

这时如果应用业务压力小了，访问数据库的并发少了，每30秒DBCP的工作线程检查连接(默认每次查3条)是否空闲达到30分钟(这是默认值)，**若连接空闲达到30分钟(minEvictableIdleTimeMillis )，就销毁连接。这时连接数减少了，呈下降趋势，将从maxIdle走向minIdle**。当小于minIdle值时，则DBCP创建新连接已使数量稳定在minIdle，并进行着新老更替。

配置initialSize=“10”时，tomcat一启动就创建10条连接。其它同上。

minIdle要与timeBetweenEvictionRunsMillis配合使用才有用,单独使用minIdle不会起作用。

---

## 参考
[DBCP连接池TestOnBorrow的坑](https://blog.csdn.net/wangyangzhizhou/article/details/52209336)      

[BasicDataSource Configuration Parameters](http://commons.apache.org/proper/commons-dbcp/configuration.html)    

[Apache Tomcat 8--The Tomcat JDBC Connection Pool](https://tomcat.apache.org/tomcat-8.0-doc/jdbc-pool.html)

https://www.oschina.net/question/219875_2143124    
http://soberchina.iteye.com/blog/2355289      
http://elf8848.iteye.com/blog/1931778     

