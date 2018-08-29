## jdbc-getConnections()源码分析
源码位置： tomcat-jdbc:8.5.5
```
org.apache.tomcat.jdbc.pool.ConnectionPool.java
```

### getConnection()
```
    /**
     * Borrows a connection from the pool. If a connection is available (in the
     * idle queue) or the pool has not reached {@link PoolProperties#maxActive
     * maxActive} connections a connection is returned immediately. If no
     * connection is available, the pool will attempt to fetch a connection for
     * {@link PoolProperties#maxWait maxWait} milliseconds.
     * @param username The user name to use for the connection
     * @param password The password for the connection
     * @return Connection - a java.sql.Connection/javax.sql.PooledConnection
     *         reflection proxy, wrapping the underlying object.
     * @throws SQLException
     *             - if the wait times out or a failure occurs creating a
     *             connection
     */
    public Connection getConnection(String username, String password) throws SQLException {
        // check out a connection
        PooledConnection con = borrowConnection(-1, username, password);
        return setupConnection(con);
    }
```

### 相关属性
```
    /**
     * Carries the size of the pool, instead of relying on a queue implementation
     * that usually iterates over to get an exact count
     */
    private AtomicInteger size = new AtomicInteger(0);


    /**
     * Contains all the idle connections
     */
    private BlockingQueue<PooledConnection> idle;


    /**
     * Contains all the connections that are in use
     * TODO - this shouldn't be a blocking queue, simply a list to hold our objects
     */
    private BlockingQueue<PooledConnection> busy;
```

### 从pools中获取Connection的实现
1. 检查连接池是否关闭
2. 尝试从idle中获取Connection，成功则返回，失败则继续执行；
3. 判断Connection数是否达到 `maxActive`个，没有的话则创建新Connection；
4. 若Connection个数已经达到`maxActive`个，则等待`MaxWait`时间后再从idle队列中获取Connection，并增加等待个数计数；
5. 超时等待后若还没有获取到，则抛出异常；

```
   /**
     * Thread safe way to retrieve a connection from the pool
     * @param wait - time to wait, overrides the maxWait from the properties,
     * set to -1 if you wish to use maxWait, 0 if you wish no wait time.
     * @param username The user name to use for the connection
     * @param password The password for the connection
     * @return a connection
     * @throws SQLException Failed to get a connection
     */
    private PooledConnection borrowConnection(int wait, String username, String password) throws SQLException {

        if (isClosed()) {
            throw new SQLException("Connection pool closed.");
        } //end if

        //get the current time stamp
        long now = System.currentTimeMillis();
        //see if there is one available immediately
        PooledConnection con = idle.poll();

        while (true) {
            if (con!=null) {
                //configure the connection and return it
                PooledConnection result = borrowConnection(now, con, username, password);
                if (result!=null) return result;
            }

            //if we get here, see if we need to create one
            //this is not 100% accurate since it doesn't use a shared
            //atomic variable - a connection can become idle while we are creating
            //a new connection
            if (size.get() < getPoolProperties().getMaxActive()) {
                //atomic duplicate check
                if (size.addAndGet(1) > getPoolProperties().getMaxActive()) {
                    //if we got here, two threads passed through the first if
                    size.decrementAndGet();
                } else {
                    //create a connection, we're below the limit
                    return createConnection(now, con, username, password);
                }
            } //end if

            //calculate wait time for this iteration
            long maxWait = wait;
            //if the passed in wait time is -1, means we should use the pool property value
            if (wait==-1) {
                maxWait = (getPoolProperties().getMaxWait()<=0)?Long.MAX_VALUE:getPoolProperties().getMaxWait();
            }

            long timetowait = Math.max(0, maxWait - (System.currentTimeMillis() - now));
            waitcount.incrementAndGet();
            try {
                //retrieve an existing connection
                con = idle.poll(timetowait, TimeUnit.MILLISECONDS);
            } catch (InterruptedException ex) {
                if (getPoolProperties().getPropagateInterruptState()) {
                    Thread.currentThread().interrupt();
                }
                SQLException sx = new SQLException("Pool wait interrupted.");
                sx.initCause(ex);
                throw sx;
            } finally {
                waitcount.decrementAndGet();
            }
            if (maxWait==0 && con == null) { //no wait, return one if we have one
                if (jmxPool!=null) {
                    jmxPool.notify(org.apache.tomcat.jdbc.pool.jmx.ConnectionPool.POOL_EMPTY, "Pool empty - no wait.");
                }
                throw new PoolExhaustedException("[" + Thread.currentThread().getName()+"] " +
                        "NoWait: Pool empty. Unable to fetch a connection, none available["+busy.size()+" in use].");
            }
            //we didn't get a connection, lets see if we timed out
            if (con == null) {
                if ((System.currentTimeMillis() - now) >= maxWait) {
                    if (jmxPool!=null) {
                        jmxPool.notify(org.apache.tomcat.jdbc.pool.jmx.ConnectionPool.POOL_EMPTY, "Pool empty - timeout.");
                    }
                    throw new PoolExhaustedException("[" + Thread.currentThread().getName()+"] " +
                        "Timeout: Pool empty. Unable to fetch a connection in " + (maxWait / 1000) +
                        " seconds, none available[size:"+size.get() +"; busy:"+busy.size()+"; idle:"+idle.size()+"; lastwait:"+timetowait+"].");
                } else {
                    //no timeout, lets try again
                    continue;
                }
            }
        } //while
    }
```

### borrowConnection：获取一个Connection
将Connection从idle队列存入到busy队列。

1. 判断Connection是否已经释放，判断是否需要reconnection
2. `MaxAge`属性：当超过maxAge毫秒后，会强制重新连接；
3. 将该Connection存入busy队列；

```
   /**
     * Validates and configures a previously idle connection
     * @param now - timestamp
     * @param con - the connection to validate and configure
     * @param username The user name to use for the connection
     * @param password The password for the connection
     * @return a connection
     * @throws SQLException if a validation error happens
     */
    protected PooledConnection borrowConnection(long now, PooledConnection con, String username, String password) throws SQLException {
        //we have a connection, lets set it up

        //flag to see if we need to nullify
        boolean setToNull = false;
        try {
            con.lock();
            if (con.isReleased()) {
                return null;
            }

            //evaluate username/password change as well as max age functionality
            boolean forceReconnect = con.shouldForceReconnect(username, password) || con.isMaxAgeExpired();

            if (!con.isDiscarded() && !con.isInitialized()) {
                //here it states that the connection not discarded, but the connection is null
                //don't attempt a connect here. It will be done during the reconnect.
                forceReconnect = true;
            }

            if (!forceReconnect) {
                if ((!con.isDiscarded()) && con.validate(PooledConnection.VALIDATE_BORROW)) {
                    //set the timestamp
                    con.setTimestamp(now);
                    if (getPoolProperties().isLogAbandoned()) {
                        //set the stack trace for this pool
                        con.setStackTrace(getThreadDump());
                    }
                    if (!busy.offer(con)) {
                        log.debug("Connection doesn't fit into busy array, connection will not be traceable.");
                    }
                    return con;
                }
            }
            //if we reached here, that means the connection
            //is either has another principal, is discarded or validation failed.
            //we will make one more attempt
            //in order to guarantee that the thread that just acquired
            //the connection shouldn't have to poll again.
            try {
                con.reconnect();
                int validationMode = getPoolProperties().isTestOnConnect() || getPoolProperties().getInitSQL()!=null ?
                    PooledConnection.VALIDATE_INIT :
                    PooledConnection.VALIDATE_BORROW;

                if (con.validate(validationMode)) {
                    //set the timestamp
                    con.setTimestamp(now);
                    if (getPoolProperties().isLogAbandoned()) {
                        //set the stack trace for this pool
                        con.setStackTrace(getThreadDump());
                    }
                    if (!busy.offer(con)) {
                        log.debug("Connection doesn't fit into busy array, connection will not be traceable.");
                    }
                    return con;
                } else {
                    //validation failed.
                    throw new SQLException("Failed to validate a newly established connection.");
                }
            } catch (Exception x) {
                release(con);
                setToNull = true;
                if (x instanceof SQLException) {
                    throw (SQLException)x;
                } else {
                    SQLException ex  = new SQLException(x.getMessage());
                    ex.initCause(x);
                    throw ex;
                }
            }
        } finally {
            con.unlock();
            if (setToNull) {
                con = null;
            }
        }
    }
```

### createConnection()：创建一个新Connection
创建一个Connection，并将其存入到 busy 队列中；
```
  /**
   * Creates a JDBC connection and tries to connect to the database.
   * @param now timestamp of when this was called
   * @param notUsed Argument not used
   * @param username The user name to use for the connection
   * @param password The password for the connection
   * @return a PooledConnection that has been connected
   * @throws SQLException Failed to get a connection
   */
  protected PooledConnection createConnection(long now, PooledConnection tUsed, String username, String password) throws SQLException {
      //no connections where available we'll create one
      PooledConnection con = create(false);
      if (username!=null) con.getAttributes().put(oledConnection.PROP_USER, username);
      if (password!=null) con.getAttributes().put(oledConnection.PROP_PASSWORD, password);
      boolean error = false;
      try {
          //connect and validate the connection
          con.lock();
          con.connect();
          if (con.validate(PooledConnection.VALIDATE_INIT)) {
              //no need to lock a new one, its not contented
              con.setTimestamp(now);
              if (getPoolProperties().isLogAbandoned()) {
                  con.setStackTrace(getThreadDump());
              }
              if (!busy.offer(con)) {
                  log.debug("Connection doesn't fit into busy array, nnection will not be traceable.");
              }
              return con;
          } else {
              //validation failed, make sure we disconnect
              //and clean up
              throw new SQLException("Validation Query Failed, enable gValidationErrors for more details.");
          } //end if
      } catch (Exception e) {
          error = true;
          if (log.isDebugEnabled())
              log.debug("Unable to create a new JDBC connection.", e);
          if (e instanceof SQLException) {
              throw (SQLException)e;
          } else {
              SQLException ex = new SQLException(e.getMessage());
              ex.initCause(e);
              throw ex;
          }
      } finally {
          // con can never be null here
          if (error ) {
              release(con);
          }
          con.unlock();
      }//catch
  }
```