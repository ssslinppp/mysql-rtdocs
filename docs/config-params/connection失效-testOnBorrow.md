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

## 参考
[DBCP连接池TestOnBorrow的坑](https://blog.csdn.net/wangyangzhizhou/article/details/52209336)      

[BasicDataSource Configuration Parameters](http://commons.apache.org/proper/commons-dbcp/configuration.html)    

[Apache Tomcat 8--The Tomcat JDBC Connection Pool](https://tomcat.apache.org/tomcat-8.0-doc/jdbc-pool.html)

https://www.oschina.net/question/219875_2143124    
http://soberchina.iteye.com/blog/2355289     


