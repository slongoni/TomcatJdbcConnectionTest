# TomcatJdbcConnectionTest (WIP)
The goal of this web app is to show how to analyze the tomcat connection pool in case of abandoned connections.

## How it works
This web app let you create as many abandoned connections as you want just clicking on a button. In fact, each time the button is pressed a new connection is opened and never closed. This scenario let us investigate how set up the tomcat datasource configuration in order to show the code line where the connection was abandoned.

## Hypotesys
You need the following tools to execute the "experiment". The tool versions are not stricly important but they are what I actualy tested:
- Tomcat 7.0.70
- Mysql Connector Java 5.1.39
- Mysql Server 5.x

## Thesis
When a jdbc connection is marked as 'abandoned' a log is shown and a jmx notification is triggered.


## Demostration
In order to demostrate the thesis you need to follow this steps:


+ Clone this repository
+ Change the /META-INF/context.xml filling in the \<Resource\> the username, the password and the jdbc url of your db under test

```xml
  <Resource name="jdbc/backoffice"
            auth="Container"
            type="javax.sql.DataSource"
            factory="org.apache.tomcat.jdbc.pool.DataSourceFactory"
            validationQuery="/* ping */"
            maxActive="32"
            minIdle="0"
            maxIdle="4"
            maxWait="3000"
            driverClassName="com.mysql.jdbc.Driver"
            username="<change_username_on_context_xml>" password="<change_password_on_context_xml>"
            url="<change_url_on_context_xml>" /> 
```

+ Deploy and run the application on your tomcat
+ Open jconsole and connect through jmx to the tomcat. Open the jmx MBean *Catalina -> DataSource -> /JdbcTomcatConnectionTest -> localhost -> javax.sql.DataSource -> jdbc/backoffice* and open the 'active' graph clicking on its value.

![alt text](https://github.com/gnosly/JdbcTomcatConnectionTest/blob/master/src/main/doc/jconsole_mbean.png "MBean opened in jconsole")

+ open in a browser the welcome page [http://localhost:8080/JdbcTomcatConnectionTest/] (http://localhost:8080/JdbcTomcatConnectionTest/) and click on the button *open a new abandoned connection*. 

![alt text](https://github.com/gnosly/JdbcTomcatConnectionTest/blob/master/src/main/doc/webapp_welcome_page.png "Web app welcome page")

+ Now you could see on jconsole that, each time you click on the button, the line of active connections grows and never goes down

![alt text](https://github.com/gnosly/JdbcTomcatConnectionTest/blob/master/src/main/doc/active_connection_increase.png "Active connections increased on jconsole") 

+ "wait..but..I don't see any warning in the log! Is the abandoned connection recognizer active by default on Tomcat?" Actually no. From the [tomcat documentation regarding the jdbc pool](https://tomcat.apache.org/tomcat-7.0-doc/jdbc-pool.html) we could use three useful properties:

| property name| description |
| --- | --- |
| removeAbandoned | (boolean) Flag to remove abandoned connections if they exceed the removeAbandonedTimeout. If set to true a connection is considered abandoned and eligible for removal if it has been in use longer than the removeAbandonedTimeout Setting this to true can recover db connections from applications that fail to close a connection. See also logAbandoned The default value is false. |
| removeAbandonedTimeout | (int) Timeout in seconds before an abandoned(in use) connection can be removed. The default value is 60 (60 seconds). The value should be set to the longest running query your applications might have.|
|logAbandoned | (boolean) Flag to log stack traces for application code which abandoned a Connection. Logging of abandoned Connections adds overhead for every Connection borrow because a stack trace has to be generated. The default value is false.|

So try to add the following properties in the \<Resource\> inside the /META-INF/context.xml as below

```xml
  <Resource name="jdbc/backoffice"
...
	removeAbandoned="true"
	removeAbandonedTimeout="10" 
	logAbandoned="true"

...	/>
```
*logAbandoned* let the tomcat to print the stacktrace of the code that is resposanble of the abandoned connection. *removeAbandonedTimeout* is set to 10 seconds for speed up the test. *removeAbandoned* is in charge of 
   1. enabling abandoned connections check and, if *logAbandoned* is true, logging the stacktrace
   2. closing really the connection

__It's important to know that without *removeAbandoned=true* the stacktrace will not appear because actually the abandoned connection check is not performed__

 
+ change the context.xml as suggested before, __restart__ the tomcat and connect again with jconsole
+ click on the *open a new abandoned connection* button as before and take a look on jconsole. Now the line grows and after 10 seconds drops down. If you take a look at the log you shoud see the stacktrace that give you the point where the connection was opened.
 
```
 [Wed Aug 31 17:24:33 CEST 2016] WARNING: [org.apache.tomcat.jdbc.pool.ConnectionPool] Connection has been abandoned PooledConnection[com.mysql.jdbc.JDBC4Connection@173432e9]:java.lang.Exception
	at org.apache.tomcat.jdbc.pool.ConnectionPool.getThreadDump(ConnectionPool.java:1072)
	at org.apache.tomcat.jdbc.pool.ConnectionPool.createConnection(ConnectionPool.java:715)
	at org.apache.tomcat.jdbc.pool.ConnectionPool.borrowConnection(ConnectionPool.java:644)
	at org.apache.tomcat.jdbc.pool.ConnectionPool.getConnection(ConnectionPool.java:187)
	at org.apache.tomcat.jdbc.pool.DataSourceProxy.getConnection(DataSourceProxy.java:128)
	at com.fgiovannetti.FireConnector.execute(FireConnector.java:29)
```

+ Now we have to demostrate that when the connection is abandoned a new jmx notificaton is triggered. First of all we have to add a new property in our \<Resource\>
```xml
  <Resource name="jdbc/backoffice"
...
	jmxEnabled="true"
...	/>
```
With *jmxEnabled* a new MBean will be registered on jmx called *tomcat.jdbc*.

+ __Restart the tomcat__ and connect again with jconsole. Go into the path *tomcat.jdbc -> ConnectionPool -> jdbc/backoffice -> /JdbcTomcatConnectionTest -> Catalina -> localhost -> org.apache.tomcat.jdbc.pool.jmx.ConnectionPool* and click on *Notifications* and therefore on *Subscribe*. 

![alt text](https://github.com/gnosly/JdbcTomcatConnectionTest/blob/master/src/main/doc/jmx_notification_subscribe.png "Jmx notification subscription") 

+ Now click on the *open a new abandoned connection* button as before and take a look on jconsole. Some notifications show up. The message field gives you the stacktrace as in the log.

![alt text](https://github.com/gnosly/JdbcTomcatConnectionTest/blob/master/src/main/doc/abandoned_connection_notifications.png "Abandoned connections notifications") 

You could think that *tomcat.jdbc -> ConnectionPool -> jdbc/backoffice -> /JdbcTomcatConnectionTest -> Catalina -> localhost -> org.apache.tomcat.jdbc.pool.jmx.ConnectionPool* is very similar to the previously analyzed *Catalina -> DataSource -> /JdbcTomcatConnectionTest -> localhost -> javax.sql.DataSource -> jdbc/backoffice*. They are almost the same, but for some reason the second MBean doesn't show any notification event.


