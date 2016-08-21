---
layout: post
title:  "Embedded Derby - Data Access - Part 3/3"
date:   2016-08-20 15:30:00 -0600
categories: java jdbc derby tutorial
customjs:
 - https://ajax.googleapis.com/ajax/libs/jquery/3.1.0/jquery.min.js
---

Goto [Part 1], [Part 2].

In this tutorial we build a data access object (DAO) to connect with our Embedded Derby
database. In the previous [Part 2], we developed a pooled connection manager and 
some basic machinery to initialize connections to our database.

[Download the code]

Although a pooled connection manager may be overkill for this small demonstration,
the same approach can be applied to larger multi-threaded applications.

First, let's take a look our final desired `main()` method.

{% highlight java linenos %}
public static void main(String[] args) {

    // Load Properites
    final Properties personsDAOprops = new Properties();
    try {
        personsDAOprops.load(
            DerbySite3.class.getResourceAsStream("PersonsDAO.properties"));
    } catch (IOException ex) {
        LOG.log(Level.SEVERE, null, ex);
        return;
    }

    // Initialize the DAO based on external configuration
    final PersonsDAO personsDAO;
    try {
        personsDAO = PersonsDAO.Build(personsDAOprops);
    } catch (ClassNotFoundException | IllegalArgumentException | IllegalStateException 
                | InstantiationException | IllegalAccessException ex) {
        LOG.log(Level.SEVERE, null, ex);
        return;
    }

    // Implement Business Logic
    personsDAO.create(
        new Person(0, "Bob", "Smith", LocalDate.of(2000, Month.MARCH, 01)));
    personsDAO.create(
        new Person(0, "Melody", "Wood", LocalDate.of(1984, Month.JULY, 23)));

    System.out.println("\n----------PERSONS----------");
    personsDAO.find("", 1000, 0).stream().forEach((p) -> {
        System.out.println(p.toString());
    });
    System.out.println("---------------------------\n");

}
{% endhighlight %}
**Output**
{% highlight shell %}
Aug 20, 2016 7:48:56 PM net.nf1198.derbysite3.util.DerbyUtil LoadDerbyEmbeddedDriver
INFO: Loaded org.apache.derby.jdbc.EmbeddedDriver

----------PERSONS----------
Person{ID=1, FirstName=Bob, LastName=Smith, DOB=2000-03-01}
Person{ID=2, FirstName=Melody, LastName=Wood, DOB=1984-07-23}
---------------------------
{% endhighlight %}

The main idea is that we've completely abstracted the access to our database. 
The `main()` method contains no remaining hints that're were using even using Derby (or even JDBC).

In the _`Load Properties`_ section, we populate a properties object using a resource file.
The body contains a single definition that defines the PersonsDAO implementation to
load into the application. In this case the properties file contains a single line:
{% highlight properties %}
CLASS=net.nf1198.derbysite3.dao.impl.EmbeddedDerbyPersonsDAO
{% endhighlight %}

Next, we _`Initialize the DAO`_ based on the external configuration. The PersonsDAO
interface provides a static `Build()` method that loads and configures the requested
implementation.

Finally, we implement our business logic. We can use the configured DAO throughout
our application. In our case, the DAO is thread-safe as we've backed it with a 
connection pool.

## Basic Setup

**Prerequsites**

* Ensure `derby-x.x.x.x.jar` is on your classpath, or included as a dependency in your project.
* Ensure `commons-dbcp2-x.x.x.jar` is on your classpath, or included as a dependency in your project.
* Ensure `commons-pool2-x.x.x.jar` is on your classpath, or included as a dependency in your project.
* Ensure `commons-logging-x.x.jar` is on your classpath, or included as a dependency in your project.

## Step 1: Project Structure

In comparison to [Part 1] and [Part 2], our final project is significantly more complex.
We've divided our classes into packages based on functionality, but in the process have
made the code easier to read and maintain.

The following screenshot shows our updated project structure:

**net.nf1198.data** contains basic data objects related to our data model, typically 
implemented as [POJOs](https://en.wikipedia.org/wiki/Plain_Old_Java_Object).

**net.nf1198.dao** contains interfaces for our [Data Access Object] (DAO) layer.

**net.nf1198.dao.impl** contains implementations of our DAO interfaces.

![Project Layout](/assets/derbysite3-projectlayout.png)

## Data Access Object Interface

The PersonsDAO interface defines the standard create-read-update-delete (CRUD) operations
as well as methods to find matching queries and count matching results. A static `Build()` 
method allows us to instantiate DAO implementations at runtime, based on a class name.

## Data Access Object Implementation

Since our application will be built around an embedded derby database, we need to create 
a corresponding implementation of the PersonsDAO. We also want to reuse the connection
pool that we developed in [Part 2]. 

Our solution in divided into three main classes, the connection pool, the database manager, 
and the actual DAO implementation.

**EmbeddedDerbyConnectionPool** wraps our connection pool into a singleton class that 
provides a single pooled [DataSource]. This class also includes a static initializer that
loads the JDBC embedded Derby driver (with a call to`DerbyUtil.LoadDerbyEmbeddedDriver()`).

**EmbeddedDerbyDBManager** contains all of the database specific methods for maintaining
our application's database (schema management).

**EmbeddedDerbyPersonsDAO** is the implementation of our PersonsDAO interface. All
Persons table database related code is located here.

## Putting it all together

### PersonsDAO Interface

{% highlight java linenos %}
public interface PersonsDAO {
    
    public Person create(Person p);
    
    public Person read(Integer id) throws IllegalArgumentException;
    
    public Person update(Person p);
    
    public void delete(Person p);
    
    public List<Person> find(String query, int limit, int offset);
    
    public long count();
    
    public long count(String query);
    
    public PersonsDAO configure(Properties properties) throws IllegalArgumentException, IllegalStateException;
    
    /**
     *
     * @param properties (Must contain at least one string-typed "CLASS" property specifying
     *                    the fully qualified class name of the PersonsDAO implementation)
     * @return A fully configured PersonsDAO instance
     * @throws ClassNotFoundException
     * @throws IllegalArgumentException
     * @throws IllegalStateException
     * @throws InstantiationException
     * @throws IllegalAccessException
     */
    public static PersonsDAO Build(Properties properties) throws ClassNotFoundException, 
          IllegalArgumentException, IllegalStateException, InstantiationException, IllegalAccessException {
        final String classID = properties.getProperty("CLASS");
        if (classID == null) {
            throw new IllegalArgumentException("DAO implementation \"CLASS\" 
                                                not specified in configuration properties");
        }
        try {
            @SuppressWarnings("unchecked")
            Class<? extends PersonsDAO> daoImplClass = 
                      (Class<? extends PersonsDAO>)Class.forName(classID);
            PersonsDAO dao = daoImplClass.newInstance();
            dao.configure(properties);
            return dao;
        } catch (ClassCastException e) {
            throw new IllegalArgumentException("Specified \"CLASS\" does not 
                                                implement PersonsDAO interface");
        }
    }
}
{% endhighlight %}

### EmbeddedDerbyPersonsDAO

{% highlight java linenos %}
public class EmbeddedDerbyPersonsDAO implements PersonsDAO {

    private static final Logger LOG = Logger.getLogger(EmbeddedDerbyPersonsDAO.class.getName());

    static {
        EmbeddedDerbyDBManager.dbUpgrade();
    }

    private Connection getConnection() throws SQLException {
        return EmbeddedDerbyConnectionPool.getDefault().getConnection();
    }

    @Override
    public Person create(final Person p) {
        final String STMT = "INSERT INTO " + PERSONS_TABLE + "(FIRSTNAME, LASTNAME, DOB) VALUES(?, ?, ?)";
        try (final Connection c = getConnection();
                final PreparedStatement s = c.prepareStatement(STMT, 1);) {
            s.clearParameters();
            s.setString(1, p.getFirstName());
            s.setString(2, p.getLastName());
            s.setDate(3, java.sql.Date.valueOf(p.getDOB()));
            final long createdID = s.executeUpdate();
            s.getConnection().commit();
            p.setID(createdID);
            return p;
        } catch (SQLException ex) {
            LOG.log(Level.SEVERE, null, ex);
        }
        return p;
    }

    @Override
    public Person read(Integer id) throws IllegalArgumentException {
        final String STMT = "SELECT ID, LASTNAME, FIRSTNAME, DOB FROM " + PERSONS_TABLE + " WHERE ID=?";
        try (final Connection c = getConnection();
                final PreparedStatement s = c.prepareStatement(STMT);) {
            s.setInt(1, id);
            try (ResultSet result = s.executeQuery()) {
                if (result.next()) {
                    final int ID = result.getInt("ID");
                    final String lastName = result.getString("LASTNAME");
                    final String firstName = result.getString("FIRSTNAME");
                    final LocalDate dob = result.getDate("DOB").toLocalDate();
                    return new Person(ID, firstName, lastName, dob);
                } else {
                    throw new IllegalArgumentException("Specified record does not exist");
                }
            }
        } catch (SQLException ex) {
            LOG.log(Level.SEVERE, null, ex);
        }
        return new Person();
    }

    @Override
    public Person update(Person p) {
        final String STMT = "UPDATE " + PERSONS_TABLE + " SET FIRSTNAME=?, LASTNAME=?, DOB=? WHERE ID=?";
        try (final Connection c = getConnection();
                final PreparedStatement s = c.prepareStatement(STMT);) {
            s.clearParameters();
            s.setString(1, p.getFirstName());
            s.setString(2, p.getLastName());
            s.setDate(3, java.sql.Date.valueOf(p.getDOB()));
            s.setLong(4, p.getID());
            s.executeUpdate();
            s.getConnection().commit();
            return p;
        } catch (SQLException ex) {
            LOG.log(Level.SEVERE, null, ex);
        }
        return p;
    }

    @Override
    public void delete(Person p) {
        final String STMT = "DELETE FROM " + PERSONS_TABLE + " WHERE ID=?";
        try (final Connection c = getConnection();
                final PreparedStatement s = c.prepareStatement(STMT);) {
            s.clearParameters();
            s.setLong(1, p.getID());
            s.executeUpdate();
            s.getConnection().commit();
        } catch (SQLException ex) {
            LOG.log(Level.SEVERE, null, ex);
        }
    }

    @Override
    public List<Person> find(String query, int limit, int offset) {
        final String STMT = "SELECT ID, FIRSTNAME, LASTNAME, DOB FROM (SELECT ID, FIRSTNAME, LASTNAME, LASTNAME_UPPER, DOB FROM "
                + PERSONS_TABLE
                + " WHERE LASTNAME_UPPER LIKE ? UNION SELECT ID, FIRSTNAME, LASTNAME, LASTNAME_UPPER, DOB FROM "
                + PERSONS_TABLE
                + " WHERE FIRSTNAME_UPPER LIKE ?) AS TMP ORDER BY LASTNAME_UPPER OFFSET ? ROWS FETCH NEXT ? ROWS ONLY";

        final String searchStr = query.toUpperCase().replace("!", "!!").replace("%", "!%").replace("_", "!_")
                .replace("[", "![") + "%";

        final List<Person> persons = new ArrayList<>();

        try (final Connection c = getConnection();
                final PreparedStatement s = c.prepareStatement(STMT);) {
            s.clearParameters();
            s.setString(1, searchStr);
            s.setString(2, searchStr);
            s.setInt(3, offset);
            s.setInt(4, limit);

            try (final ResultSet result = s.executeQuery()) {
                while (result.next()) {
                    final int ID = result.getInt("ID");
                    final String lastName = result.getString("LASTNAME");
                    final String firstName = result.getString("FIRSTNAME");
                    final LocalDate dob = result.getDate("DOB").toLocalDate();
                    Person p = new Person(ID, firstName, lastName, dob);
                    persons.add(p);
                }
            } catch (SQLException e) {
                LOG.log(Level.SEVERE, "Error searching persons.", e);
            }
        } catch (SQLException ex) {
            LOG.log(Level.SEVERE, null, ex);
        }
        return persons;
    }

    @Override
    public long count() {
        final String STMT = "SELECT COUNT(*) FROM " + PERSONS_TABLE;
        try (final Connection c = getConnection();
                final PreparedStatement s = c.prepareStatement(STMT);) {
            try (final ResultSet result = s.executeQuery()) {
                if (result.next()) {
                    return result.getInt(1);
                }
            } catch (SQLException e) {
                LOG.log(Level.SEVERE, "Error counting persons", e);
            }
        } catch (SQLException ex) {
            LOG.log(Level.SEVERE, null, ex);
        }
        return 0;
    }

    @Override
    public long count(String query) {
        final String STMT = "SELECT COUNT(*) FROM (SELECT ID FROM "
                + PERSONS_TABLE + " WHERE LASTNAME_UPPER LIKE ? UNION SELECT ID FROM " + PERSONS_TABLE
                + " WHERE FIRSTNAME_UPPER LIKE ?) AS TMP";

        final String searchStr = query.toUpperCase().replace("!", "!!").replace("%", "!%").replace("_", "!_")
                .replace("[", "![") + "%";

        try (final Connection c = getConnection();
                final PreparedStatement s = c.prepareStatement(STMT);) {
            try {
                s.setString(1, searchStr);
                s.setString(2, searchStr);

                try (ResultSet result = s.executeQuery()) {
                    if (result.next()) {
                        return result.getInt(1);
                    }
                }
            } catch (SQLException e) {
                LOG.log(Level.SEVERE, "Error executing person search", e);
            }
        } catch (SQLException ex) {
            LOG.log(Level.SEVERE, null, ex);
        }
        return 0;
    }

    @Override
    public PersonsDAO configure(Properties properties) throws IllegalArgumentException, IllegalStateException {
        // Nothing to configure
        return this;
    }
}
{% endhighlight %}

### EmbeddedDerbyDBManager 

{% highlight java linenos %}
public final class EmbeddedDerbyDBManager {

    private static final Logger LOG = Logger.getLogger(EmbeddedDerbyDBManager.class.getName());

    final public static String DBINFO_TABLE = "DBINFO"; // Don't change this value
    // unless you also create a
    // dbUpgrade version
    // migration!

    final public static String PERSONS_TABLE = "PERSONS"; // Don't change this value
    // unless you also create a
    // dbUpgrade version
    // migration!

    public static void dbUpgrade() {
        try (final Connection conn = EmbeddedDerbyConnectionPool.getDefault().getConnection();) {
            boolean done = false;
            while (!done) {
                switch (getDBVersion(conn)) {
                    case 0:
                        // migrate database from level 0 (new database) to version 1

                        try (Statement stmt = conn.createStatement()) {
                            conn.setAutoCommit(false);
                            stmt.addBatch("CREATE TABLE " + DBINFO_TABLE
                                    + "(ID INT NOT NULL GENERATED ALWAYS AS IDENTITY (START WITH 1, INCREMENT BY 1) CONSTRAINT DBINFO_PK PRIMARY KEY, K VARCHAR(50), V VARCHAR(50))");
                            stmt.addBatch("CREATE TABLE " + PERSONS_TABLE
                                    + "(ID BIGINT NOT NULL GENERATED ALWAYS AS IDENTITY CONSTRAINT PERSONS_PK PRIMARY KEY, FIRSTNAME VARCHAR(50), LASTNAME VARCHAR(50), FIRSTNAME_UPPER VARCHAR(50) GENERATED ALWAYS AS (UPPER(FIRSTNAME)), LASTNAME_UPPER VARCHAR(50) GENERATED ALWAYS AS (UPPER(LASTNAME)), DOB DATE)");
                            stmt.addBatch("CREATE INDEX PERSONS_FIRST ON PERSONS(FIRSTNAME_UPPER)");
                            stmt.addBatch("CREATE INDEX PERSONS_LAST ON PERSONS(LASTNAME_UPPER)");
                            stmt.addBatch("INSERT INTO " + DBINFO_TABLE + " (K, V) VALUES ('VERSION', '1')");
                            stmt.executeBatch();
                            conn.commit();
                        } catch (SQLException e) {
                            LOG.log(Level.SEVERE, "Error initializing database (0->1)", e);
                            done = true;
                        }
                        break;
                    case 1:
                        done = true;
                        break;
                }
            }
        } catch (SQLException ex) {
            LOG.log(Level.SEVERE, null, ex);
        }
    }

    
    public static int getDBVersion(Connection conn) {

        int version = -1; // version undefined

        // check if dbInfo table exists, if so check the version
        if (JDBCUtil.getTables(conn).contains(DBINFO_TABLE)) {
            try (final Statement stmt = conn.createStatement();
                    final ResultSet result = stmt
                    .executeQuery("SELECT V FROM " + DBINFO_TABLE + " WHERE K='VERSION'")) {
                if (result.next()) {
                    version = Integer.parseInt(result.getString("V"));
                }
            } catch (SQLException | NumberFormatException e) {
                LOG.log(Level.SEVERE, "Error retrieving database version", e);
            }
        } else {
            version = 0;
        }

        return version;
    }
}
{% endhighlight %}

## Complete code

[Download the Code]

[connection pool]: https://en.wikipedia.org/wiki/Connection_pool
[Apache DBCP2]: https://commons.apache.org/proper/commons-dbcp/
[JDBC]: https://en.wikipedia.org/wiki/Java_Database_Connectivity
[Data Access Object]: https://en.wikipedia.org/wiki/Data_access_object
[Data Source]: https://docs.oracle.com/javase/7/docs/api/javax/sql/DataSource.html
[Apache Derby]: https://db.apache.org/derby/manuals/index.html
[Driver Manager]: https://docs.oracle.com/javase/8/docs/api/java/sql/DriverManager.html
[Part 1]: {% post_url 2016-08-15-DerbyEmbedded-Part1 %}
[Part 2]: {% post_url 2016-08-19-DerbyEmbedded-Part2 %}
[Download the Code]:/assets/derbysite3-dist.tar.gz