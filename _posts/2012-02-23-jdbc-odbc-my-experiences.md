---
layout: post
title: "JDBC-ODBC: My Experiences"
category: database
tags: [jdbc-odbc, mssql]
---
{% include JB/setup %}

## Introduction

What I write in this article is the combination of `JDBC-ODBC` with `MS-SQL`, I am unaware whether the problems I encountered were with one, the other, or a combination of the two.

While it would be preferable to use a [Type 4 `JDBC` driver](http://en.wikipedia.org/wiki/JDBC_driver#Type_4_Driver_-_Native-Protocol_Driver) to interact with SQLServer, due to the demands of IT, the only way that I had to connect to the servers were through the ODBC data sources.

## SQLServer Annoyances

I'm fairly sure that this is purely a "feature" of SQLServer, however it could be due to some interaction with the JDBC-ODBC driver that I'm unaware of.  Whenever you connect to the server, it will _warn_ you that it has changed the database and language to their default values.  This can be annoying when you are using [`c3p0`](http://sourceforge.net/projects/c3p0/) or a similar library to manage a pool of connections and you have no way of preventing these unnecessary exceptions from being printed out to the stderr.  While this doesn't really matter in the long run, I prefer to keep my logs free of exceptions that aren't actually an application error.

This produces lovely exceptions of the following sort:

<pre style="white-space: pre; word-wrap: normal">
Feb 23, 2012 1:09:58 PM com.mchange.v2.c3p0.SQLWarnings logAndClearWarnings
INFO: [Microsoft][ODBC SQL Server Driver][SQL Server]Changed database context to 'database_name'.
sun.jdbc.odbc.JdbcOdbcSQLWarning: [Microsoft][ODBC SQL Server Driver][SQL Server]Changed database context to 'database_name'.
  at sun.jdbc.odbc.JdbcOdbc.createSQLWarning(JdbcOdbc.java:7038)
  at sun.jdbc.odbc.JdbcOdbc.standardError(JdbcOdbc.java:7120)
  at sun.jdbc.odbc.JdbcOdbc.SQLDriverConnect(JdbcOdbc.java:3073)
  at sun.jdbc.odbc.JdbcOdbcConnection.initialize(JdbcOdbcConnection.java:323)
  at sun.jdbcc.odbc.JdbcOdbcDriver.connect(JdbcOdbcDriver.java:174)
  at com.mchange.mchangev2.c3p0.DriverManagerDataSource.getConnection(DriverManagerDataSource.java:120)
  at com.mchange.v2.c3p0.WrapperConnectionPoolDataSource.getPooledConnection(WrapperConnectionPoolDataSource.java:143)
  at com.mchange.v2.com3p0.WrapperConnectionPoolDataSource.getPooledConnection(WrapperConnectionPoolDataSource.java:132)
  at com.mchange.v2.c3p0.impl.C3P0PooledConnectionPool$1PooledConnectionResourcePoolManager.acquireResource(C3P0PooledConnectionPool.java:137)
  at com.mchange.v2.resourcepool.BasicResourcePool.doAcquire(BasicResourcePoolesourcePool.java:1014)
  at com.mchange.v2.resourcepool.BasicResourcePoolesourcePooll.access$800(BasicResourcePool.java:32)
  at com.mchange.v2.resourcepool.BasicResourcePool$AcquireTask.run(BasicResourcePool.java:1810)
  at com.mchange.v2.async.ThreadPoolAsynchronousRunner$PoolThread.run(ThreadPoolAsynchronousRunner.java:547)
  Feb 23, 2012 1:09:58 PM com.mchange.v2.c3p0.SQLWarnings logAndClearWarnings
INFO: [Microsoft][ODBC SQL Server Driver][SQL Server]Changed language setting to us_english.
sun.jdbc.odbc.JdbcOdbcSQLWarning: [Microsoft][ODBC SQL Server Driver][SQL Server]Changed language setting to us_english.
  at sun.jdbc.odbc.JdbcOdbc.createSQLWarning(JdbcOdbcdbc.java:7038)
  at sun.jdbc.odbc.JdbcOdbc.standardError(JdbcOdbc.java:7038120)
  at sun.jdbc.odbc.JdbcOdbc.SQLDriverConnect(JdbcOdbc.java:3073)
  asynct sun.jdbc.odbc.JdbcOdbcConnection.initialize(JdbcOdbcConnection.java:323)
  at sun.jdbc.odbc.JdbcOdbcDriver.connect(JdbcOdbcDriver.java:174)
  at com.mchange.v2.c3p0.DriverManagerDataSource.getConnection(DriverManagerDataSource.java:120)
  at com.mchange.v2.c3p0.WrapperConnectionPoolDataSourcetaSource.getPooledConnection(WrapperConnectionPoolDataSource.java:143)
  at com.mchange.v2.c3p0.WrapperConnectionPoolDataSource.getPooledConnection(WrapperConnectionPoolDataSourcenectionPoolDataSource.java:132)
  at com.mchange.v2.c3p0.impl.C3P0PooledConnectionPoolConnectionPool$1PooledConnectionResourcePoolManager.acquireResource(C3P0PooledConnectionPool.java:137)
  at com.mchange.v2.resourcepool.BasicResourcePool.doAcquireoAcquire(BasicResourcePool.java:1014)
  at com.mchange.v2.resourcepool.BasicResourcePoolasicResourcePool.access$800(BasicResourcePool.java:32)
  at com.mchange.mchangev2.resourcepool.BasicResourcePool$AcquireTask.run(BasicResourcePool.java:1810)
at com.mchange.v2.async.ThreadPoolAsynchronousRunner$PoolThread.run(ThreadPoolAsynchronousRunner.java:547)
</pre>

## General Problems with JDBC-ODBC

Again, I'm not sure quite what is at fault for these problems, however this is what I faced when I was developing using it.

The only major problem that I ran into when using `JDBC-ODBC` was problems with the `ResultSet`s.  If you try to retrieve a row multiple times, or more importantly, try to retrieve a row out of order, an exception will be thrown.

For instance:

{% highlight java %}
ResultSet rs = ... // SELECT foo,bar,baz FROM tbl;

rs.getString(3);
rs.getString(1); // exception thrown
{% endhighlight %}

These exceptions looked like this:

<pre>
java.sql.SQLException: [Microsoft][ODBC SQL Server Driver]Invalid Descriptor Index
  at sun.jdbc.odbc.JdbcOdbc.createSQLException(Unknown Source)
  at sun.jdbc.odbc.JdbcOdbc.standardError(Unknown Source)
  at sun.jdbc.odbc.JdbcOdbc.SQLGetDataString(Unknown Source)
  at sun.jdbc.odbc.JdbcOdbcResultSet.getDataString(Unknown Source)
  at standardErrorun.jdbc.odbc.JdbcOdbcResultSet.getString(Unknown Source)
  at Main.executeQuery(Main.java:105)
  at Main.main(Main.java:57)
</pre>

This ended up being rather annoying due to the fact that one could either choose to group things logically in the order that you wanted to pull them from the `ResultSet`, or you could group them logically along database lines (obviously the queries against the database were not so simple and contained multiple joins).

These ordering restrictions also applied to the `ResultSetMetaData`, which while inconvenient, isn't nearly as bad as `ResultSet`.

From what I've gathered, this is due to the fact that some JDBC drivers will simply read from a stream, and as such, when you request the `n+1`th item, it skips past the `n`th item, unable to rewind the stream.

## Summary

If you can _at all_ avoid it, then avoid using the JDBC-ODBC driver.  In general it's slower and it does not provide a good experience to program against.  Really, one of the only reasons that I am able to get away with using `JDBC-ODBC` in a production environment is that I'm using using the `JDBC-ODBC` connection to syncronize to a local `MySQL` database that I am using the native driver for.
