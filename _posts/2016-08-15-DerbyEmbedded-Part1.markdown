---
layout: post
title:  "Yet another JDBC tutorial - Part 1"
date:   2016-08-15 06:14:26 -0600
categories: java jdbc derby tutorial
customjs:
 - https://ajax.googleapis.com/ajax/libs/jquery/3.1.0/jquery.min.js
---
In this tutorial we use [JDBC] to connect to an [Apache Derby] database. In
part 2, we will develop several convenience classes for managing connections to
JDBC databases. In part 3, we will create a [Data Access Object] layer that helps 
us separate application logic from database logic.

Connecting to databases with JDBC is a little more complicated than you would expect.
This is especially true if you're used to more streamlined APIs such as those provided by MongoDB
or other noSQL databases. [Apache Derby] add additional quirks for database connection management.

Let's summarize what's required to connect to a database using JDBC and Derby.

## Basic Setup

**Prerequsites**

* Ensure `derby-x.x.x.x.jar` is on your classpath, or included as a dependency in your project.
  (Maven is a good way to find this dependency)

**Step 1**: Load the JDBC driver (i.e. "`org.apache.derby.jdbc.EmbeddedDriver`")

{% highlight java %}
final String DERBY_EMBEDDED = "org.apache.derby.jdbc.EmbeddedDriver";
try {
    Class.forName(DERBY_EMBEDDED).newInstance();
    LOG.log(Level.INFO, "Loaded {0}", DERBY_EMBEDDED);
} catch (ClassNotFoundException e) {
    ...
}
{% endhighlight %}

Calling `Class.forName(dbDriver)` loads the specified driver or throws an exception
if the driver can't be found. The driver must be available on the classpath for this to work.

Appending `.newInstance()` isn't recommended by JDBC, but is [recommended in the Derby documentation](http://db.apache.org/derby/docs/10.12/devguide/tdevdvlp20349.html) 
to guarantee Derby will be booted in any JVM.

**Step 2**: Use the `DriverManager` to open a connection to our database

{% highlight java %}
final String dbURL = "jdbc:derby:/path/to/database";
try (final Connection conn = DriverManager.getConnection(dbURL + ";create=true");
    final Statement stmt = conn.createStatement()) {
        ... use database ...
    } catch (SQLException e) {
            LOG.log(Level.SEVERE, "Unhandled SQLException:", e);
    }
{% endhighlight %}

The [Driver Manager] returns a `Connection` to the database. Use try-with-resources
to ensure that we free resources after we're done using the connection.

Review [Derby JDBC database connection URL](https://db.apache.org/derby/docs/10.12/devguide/cdevdvlp17453.html) 
and [Booting databases](http://db.apache.org/derby/docs/10.12/devguide/cdevdvlp27715.html) 
for more information about connecting to Derby databases.

**Step 3**: Unload the driver and attempt to free resources

{% highlight java %}
static void ShutdownDerby(final String dbURL) {
    boolean derbyXJ015;
    boolean derby08006;
    try {
        // note: From Java 8, deregister=false is suggested unless properly configured security manager
        //       Refer to derby docs: http://db.apache.org/derby/docs/10.12/devguide/tdevdvlp20349.html
        DriverManager.getConnection("jdbc:derby:" + dbURL + ";shutdown=true;deregister=false");
    } catch (SQLException e) {
        derbyXJ015 = e.getSQLState().equals("XJ015"); // expected state for system shutdown
        derby08006 = e.getSQLState().equals("08006"); // expected state for DB shutdown
        if (derbyXJ015) {
            LOG.log(Level.INFO, "Derby system shutdown");
        } else if (derby08006) {
            LOG.log(Level.INFO, "Derby database `{0}` shutdown", dbURL);
        } else {
            LOG.log(Level.SEVERE, "error unloading " + dbURL, e);
        }
    } finally {
        System.gc();
    }
}
{% endhighlight %}

Apache Derby uses an unconventional mechanism for notifying the application of successful 
unloading of resources. Use the `"jdbc:derby:;shutdown=true"` connection string to signal
Derby to unload, then check that the SQLState equals `"XJ015"`.

Refer to [Shutting down the system](http://db.apache.org/derby/docs/10.12/devguide/tdevdvlp20349.html),
 [Shutting down Derby or an individual database](http://db.apache.org/derby/docs/10.12/devguide/tdevdvlp40464.html), 
and [Running embedded Derby with a security manager](https://db.apache.org/derby/docs/10.12/security/csecembeddedperms.html) 
for more information about shutting down Derby databases in Java 8 and later.

Note that resources may not be freed until garbage collection has been triggered.

## Putting it all together

{% highlight java %}
public class DerbyPartOne {

    static final Logger LOG = Logger.getLogger(DerbyPartOne.class.getName());
    static final String DERBY_EMBEDDED = "org.apache.derby.jdbc.EmbeddedDriver";

    public static void main(String[] args) {
        try {
            // Load Derby Embedded Driver
            Class.forName(DERBY_EMBEDDED);
            LOG.log(Level.INFO, "Loaded {0}", DERBY_EMBEDDED);

            final String dbURL = "memory:myDB";
            final String dbPath = "jdbc:derby:" + dbURL + ";create=true";

            // Use Derby Driver
            try (final Connection conn = DriverManager.getConnection(dbPath);
                    final Statement stmt = conn.createStatement()) {
                
                /*
                USE JDBC CONNECTION HERE
                */
                
            } catch (SQLException ex) {
                LOG.log(Level.SEVERE, null, ex);
            } finally {
                // Close Database
                ShutdownDerby(dbURL);
            }
        } catch (ClassNotFoundException ex) {
            LOG.log(Level.SEVERE, null, ex);
        } finally {
            // Shutdown Derby System
            ShutdownDerby("");
        }
    }

    static void ShutdownDerby(final String dbURL) {
        boolean derbyXJ015;
        boolean derby08006;
        try {
            // note: From Java 8, deregister=false is suggested unless properly configured security manager
            //       Refer to derby docs: http://db.apache.org/derby/docs/10.12/devguide/tdevdvlp20349.html
            DriverManager.getConnection("jdbc:derby:" + dbURL + ";shutdown=true;deregister=false");
        } catch (SQLException e) {
            derbyXJ015 = e.getSQLState().equals("XJ015"); // expected state for system shutdown
            derby08006 = e.getSQLState().equals("08006"); // expected state for DB shutdown
            if (derbyXJ015) {
                LOG.log(Level.INFO, "Derby system shutdown");
            } else if (derby08006) {
                LOG.log(Level.INFO, "Derby database `{0}` shutdown", dbURL);
            } else {
                LOG.log(Level.SEVERE, "error unloading " + dbURL, e);
            }
        } finally {
            System.gc();
        }
    }
}
{% endhighlight %}

Our first pass at the code is quite ungainly, riddled with nested try-catch blocks and not-so-apparent
driver state. 

In [Part 2] of this tutorial we will develop several database management classes that simplify our 
application logic and make it easier to keep track of our business logic.

[JDBC]: https://en.wikipedia.org/wiki/Java_Database_Connectivity
[Data Access Object]: https://en.wikipedia.org/wiki/Data_access_object
[Apache Derby]: https://db.apache.org/derby/manuals/index.html
[Driver Manager]: https://docs.oracle.com/javase/8/docs/api/java/sql/DriverManager.html

[Part 2]: /part2.html