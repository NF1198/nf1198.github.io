---
layout: post
title:  "JDBC Connection Pooling Essentials - Part 2/3"
date:   2016-08-19 15:14:00 -0600
categories: java jdbc derby tutorial
customjs:
 - https://ajax.googleapis.com/ajax/libs/jquery/3.1.0/jquery.min.js
---

Goto [Part 1].

In this tutorial we use [JDBC] to connect to an [Apache Derby] database using 
a [connection pool] provided by [Apache DBCP2]. This tutorial builds on the basics
developed in [Part 1].

In all but the simplest use-cases, an application will use multiple threads to
access a shared database resource. This is especially true in a web-server type
application that handles multiple clients simultaneously. One critical performance
consideration the developer needs to make is how to manage database connections.

Typically, each thread of an application will use a distinct connection, but there is
no reason a connection can't be reused if a thread has released it. Connection
pooling adds an automated mechanism that allows applications to reuse connections.

[Apache DBCP2] (Database Connection Pools) is a convenient, embeddable connection
pooling solution.

## Basic Setup

**Prerequsites**

* Ensure `derby-x.x.x.x.jar` is on your classpath, or included as a dependency in your project.
* Ensure `commons-dbcp2-x.x.x.jar` is on your classpath, or included as a dependency in your project.
* Ensure `commons-pool2-x.x.x.jar` is on your classpath, or included as a dependency in your project.
* Ensure `commons-logging-x.x.jar` is on your classpath, or included as a dependency in your project.

**Step 1**: Create a helper class for loading and unloading the embedded derby driver.

This static utility class eases the pain of loading and unloading the derby embedded driver.

{% highlight java %}
/*
 * Utility class to help with loading and unloading the 
 * Derby embedded driver.
 */
final class DerbyDriverLoader {

    private static final Logger LOG = Logger.getLogger(DerbyDriverLoader.class.getName());
    static final String DERBY_EMBEDDED = "org.apache.derby.jdbc.EmbeddedDriver";

    private DerbyDriverLoader() {
    }

    /**
     * Loads the Derby Embedded JDBC driver
     * @return true if the driver was loaded successfully
     */
    public static boolean LoadDriver() {
        try {
            Class.forName(DERBY_EMBEDDED);
            LOG.log(Level.INFO, "Loaded {0}", DERBY_EMBEDDED);
            return true;
        } catch (ClassNotFoundException ex) {
            LOG.log(Level.SEVERE, null, ex);
        }
        return false;
    }

    /**
     * Unloads the Derby Embedded JDBC driver
     * @return true if the driver was unloaded successfully
     */
    public static boolean UnloadDriver() {
        boolean derbyXJ015;
        try {
            // note: From Java 8, deregister=false is suggested unless properly configured security manager
            //       Refer to derby docs: http://db.apache.org/derby/docs/10.12/devguide/tdevdvlp20349.html
            DriverManager.getConnection("jdbc:derby:;shutdown=true;deregister=false");
        } catch (SQLException e) {
            derbyXJ015 = e.getSQLState().equals("XJ015"); // expected state for system shutdown
            if (derbyXJ015) {
                LOG.log(Level.INFO, "Shutdown {0}", DERBY_EMBEDDED);
                return true;
            } else {
                LOG.log(Level.SEVERE, "Unexpected error unloading derby", e);
            }
        } finally {
            System.gc();
        }
        return false;
    }
}
{% endhighlight %}

**Step 2**: Create a helper method to setup a connection pool.

Review the comments in the [PoolingDataSourceExample.java](http://svn.apache.org/viewvc/commons/proper/dbcp/trunk/doc/PoolingDataSourceExample.java?view=markup)
example for further explanation.

{% highlight java %}
/*
 * Create a pooling datasource using Apache dbcp2.
 * Refer to the "PoolingDataSource" example in the dbcp2 documentation for
 * a complete explanation. ref: http://commons.apache.org/proper/commons-dbcp/index.html
 * 
 * @param connectURI
 * @return pooled dataSource
 */
public static DataSource setupPoolingDataSource(String connectURI) {
   ConnectionFactory connectionFactory
           = new DriverManagerConnectionFactory(connectURI, null);
   PoolableConnectionFactory poolableConnectionFactory
           = new PoolableConnectionFactory(connectionFactory, null);
   ObjectPool<PoolableConnection> connectionPool
           = new GenericObjectPool<>(poolableConnectionFactory);
   poolableConnectionFactory.setPool(connectionPool);
   PoolingDataSource<PoolableConnection> dataSource
           = new PoolingDataSource<>(connectionPool);
   return dataSource;
}
{% endhighlight %}

## Putting it all together

Now you can load the derby driver and use a connection pool to access resources.
In a larger application, you may have multiple threads borrowing connections from the 
pool. 

{% highlight java %}
public static void main(String[] args) {
    final String DB_NAME = "memory:myDB";
    final String DB_URL = "jdbc:derby:" + DB_NAME + ";create=true";

    try {
        DerbyDriverLoader.LoadDriver();

        DataSource dataSource = setupPoolingDataSource(DB_URL);

        try (Connection conn = dataSource.getConnection();
                Statement stmt = conn.createStatement()) {

            // USE DERBY EMBEDDED //

        } catch (SQLException ex) {
            LOG.log(Level.SEVERE, null, ex);
        } finally {

        }

    } finally {
        DerbyDriverLoader.UnloadDriver();
    }
}
{% endhighlight %}

## Complete code

{% highlight java %}
...

public class DerbyPartTwo {

    final static Logger LOG = Logger.getLogger(DerbyPartTwo.class.getName());

    public static void main(String[] args) {
        final String DB_NAME = "memory:myDB";
        final String DB_URL = "jdbc:derby:" + DB_NAME + ";create=true";

        try {
            DerbyDriverLoader.LoadDriver();

            DataSource dataSource = setupPoolingDataSource(DB_URL);

            try (Connection conn = dataSource.getConnection();
                    Statement stmt = conn.createStatement()) {

                // USE DERBY EMBEDDED //

            } catch (SQLException ex) {
                LOG.log(Level.SEVERE, null, ex);
            } finally {

            }

        } finally {
            DerbyDriverLoader.UnloadDriver();
        }
    }

    /*
     * Create a pooling datasource using Apache dbcp2.
     * Refer to the "PoolingDataSource" example in the dbcp2 documentation for
     * a complete explanation. ref: http://commons.apache.org/proper/commons-dbcp/index.html
     * 
     * @param connectURI
     * @return pooled dataSource
     */
    public static DataSource setupPoolingDataSource(String connectURI) {
        ConnectionFactory connectionFactory
                = new DriverManagerConnectionFactory(connectURI, null);
        PoolableConnectionFactory poolableConnectionFactory
                = new PoolableConnectionFactory(connectionFactory, null);
        ObjectPool<PoolableConnection> connectionPool
                = new GenericObjectPool<>(poolableConnectionFactory);
        poolableConnectionFactory.setPool(connectionPool);
        PoolingDataSource<PoolableConnection> dataSource
                = new PoolingDataSource<>(connectionPool);
        return dataSource;
    }

}

/*
 * Utility class to help with loading and unloading the 
 * Derby embedded driver.
 */
final class DerbyDriverLoader {

    private static final Logger LOG = Logger.getLogger(DerbyDriverLoader.class.getName());
    static final String DERBY_EMBEDDED = "org.apache.derby.jdbc.EmbeddedDriver";

    private DerbyDriverLoader() {
    }

    /**
     * Loads the Derby Embedded JDBC driver
     *
     * @return true if the driver was loaded successfully
     */
    public static boolean LoadDriver() {
        try {
            Class.forName(DERBY_EMBEDDED);
            LOG.log(Level.INFO, "Loaded {0}", DERBY_EMBEDDED);
            return true;
        } catch (ClassNotFoundException ex) {
            LOG.log(Level.SEVERE, null, ex);
        }
        return false;
    }

    /**
     * Unloads the Derby Embedded JDBC driver
     *
     * @return true if the driver was unloaded successfully
     */
    public static boolean UnloadDriver() {
        boolean derbyXJ015;
        try {
            // note: From Java 8, deregister=false is suggested unless properly configured security manager
            //       Refer to derby docs: http://db.apache.org/derby/docs/10.12/devguide/tdevdvlp20349.html
            DriverManager.getConnection("jdbc:derby:;shutdown=true;deregister=false");
        } catch (SQLException e) {
            derbyXJ015 = e.getSQLState().equals("XJ015"); // expected state for system shutdown
            if (derbyXJ015) {
                LOG.log(Level.INFO, "Shutdown {0}", DERBY_EMBEDDED);
                return true;
            } else {
                LOG.log(Level.SEVERE, "Unexpected error unloading derby", e);
            }
        } finally {
            System.gc();
        }
        return false;
    }
}
{% endhighlight %}

[connection pool]: https://en.wikipedia.org/wiki/Connection_pool
[Apache DBCP2]: https://commons.apache.org/proper/commons-dbcp/
[JDBC]: https://en.wikipedia.org/wiki/Java_Database_Connectivity
[Data Access Object]: https://en.wikipedia.org/wiki/Data_access_object
[Apache Derby]: https://db.apache.org/derby/manuals/index.html
[Driver Manager]: https://docs.oracle.com/javase/8/docs/api/java/sql/DriverManager.html
[Part 1]: {% post_url 2016-08-15-DerbyEmbedded-Part1 %}