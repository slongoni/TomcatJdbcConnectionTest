# JdbcTomcatConnectionTest (WIP)
The goal of this web app is to show how analyze the tomcat connection pool in case of abandoned connections.

## How it works
This web app let you create as many abandoned connections as you want just clicking on the button in the welcome page. In fact, each time the button is pressed a new connection is opened and never closed. This scenario let us investigate how set up the tomcat datasource configuration in order to show the peace of code that abandoned the connection.

## Hypotesys
You need the following tools to execute the "experiment". The tool versions are not stricly important but they are what I actualy tested:
- Tomcat 7.0.70
- Mysql Connector Java 5.1.39
- Mysql Server 5.x

## Thesis
When a jdbc connection is marked as 'abandoned' a log is shown and a jmx notification is triggered.


## Demostration
In order to demostrate the thesis you need to follow this steps:


1. clone this repository
2. change the /META-INF/context.xml filling the resource tag with the username, the password and the jdbc url of your db under test

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

3. deploy and run the application on your tomcat
4. open jconsole and connect through jmx to the tomcat. Open the jmx MBean Catalina -> DataSource -> /JdbcTomcatConnectionTest -> localhost -> javax.sql.DataSource -> jdbc/backoffice and show the metric 'active' clicking on the value of it.

![alt text](https://github.com/gnosly/JdbcTomcatConnectionTest/blob/master/src/main/doc/jconsole_mbean.png "MBean opened in jconsole")

5. open in a browser the web app welcome page http://localhost:8080/JdbcTomcatConnectionTest/ and click on the button. Now you could see on Jmx that each time you click the button the line of active connection grows and never goes down   

![alt text](https://github.com/gnosly/JdbcTomcatConnectionTest/blob/master/src/main/doc/webapp_welcome_page.png "Web app welcome page")


6. wait..but..the abandoned connection recognizer is active by default on Tomcat?. Actually no. you need to add the following properties in the Resource configuration inside the context.xml

```xml
  <Resource name="jdbc/backoffice"
...
	removeAbandoned="true"
	removeAbandonedTimeout="10" 
	logAbandoned="true"

...
```


 //TODO: tomcat doc page
 //TODO: descr meaning of the properties
 
 7. change the context.xml as suggested before and restart the tomcat with the new webapp
 8. click on the button again and take a look on Jmx. Now the line grows and after 10 seconds drop down. If you see the log you shoud see the log that give you the point where the connection was opened.
 
 [Wed Aug 31 17:24:33 CEST 2016] WARNING: [org.apache.tomcat.jdbc.pool.ConnectionPool] Connection has been abandoned PooledConnection[com.mysql.jdbc.JDBC4Connection@173432e9]:java.lang.Exception
	at org.apache.tomcat.jdbc.pool.ConnectionPool.getThreadDump(ConnectionPool.java:1072)
	at org.apache.tomcat.jdbc.pool.ConnectionPool.createConnection(ConnectionPool.java:715)
	at org.apache.tomcat.jdbc.pool.ConnectionPool.borrowConnection(ConnectionPool.java:644)
	at org.apache.tomcat.jdbc.pool.ConnectionPool.getConnection(ConnectionPool.java:187)
	at org.apache.tomcat.jdbc.pool.DataSourceProxy.getConnection(DataSourceProxy.java:128)
	at com.fgiovannetti.FireConnector.execute(FireConnector.java:29)

9. jmx notificaton ( jmxEnabled )


