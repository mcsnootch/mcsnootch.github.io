---
layout: post
title: "JDBI, Oracle and null"
date: 2014-06-19 15:59:43 +0100
comments: true
categories: 
---

[JDBI](http://jdbi.org) is a pretty neat framework for a light-touch integration with JDBC in your application. 
t's SQL Object API lets you annotate an interface with straight SQL queries like so:
```java
interface FooDao {
   @SqlInsert( "insert into bar values (:foo)" )
   public void insert( @Bind("foo") String foo );
}
```
then get dbi to create you an instance of this dao which will magically prepare and execute the statements you've bound to the methods in the interface.

But try supplying null as a parameter to your dao, and you'll get a nasty surprise:

<!-- more -->

```java
    FooDao dao = dbi.open(FooDao.class);

    dao.insert(null);
```

results in a lengthy stack trace culminating in:

    Caused by: ! java.sql.SQLException: Invalid column type: 1111

Ugh.

Why is this happening? Well, under the hood JDBI is unable to work out the type of your null argument, and falls back to treating your parameter as an instance of Object, leading to a JDBI ObjectArgument being created to bind your parameter into the PreparedStatement it's building for you under the covers. When this ObjectArgument is applied to your query it does this: 

```java

public void apply(int position, PreparedStatement statement, StatementContext ctx) throws SQLException
{
   if (value != null) {
      statement.setObject(position, value);
   }
   else {
      statement.setNull(position, Types.OTHER);
   }
}

```

Our null parameter is successfully mapped to a setNull on the statement, but the sql type supplied to statement gets the Oracle driver's knickers in a twist, as it doesn't recognise ```java.sql.Types.OTHER``` as a valid sql type, and throws up at your feet. 

Fortunately, JDBI gives us an opportunity to extend it's functionality by registering somethigng with it which will set the correct sql type for the finicky Oracle driver:

```java
public class NullArgumentFactory implements ArgumentFactory<Object> {

   @Override
      public boolean accepts(Class<?> expectedType, Object value, StatementContext ctx) {
         return value == null;
      }

   @Override
      public Argument build(Class<?> expectedType, Object value, StatementContext ctx) {
         return new NullArgument();
      }

   private class NullArgument implements Argument {
      @Override
         public void apply(int position, PreparedStatement statement, StatementContext ctx) throws SQLException {
            statement.setNull(position, Types.NULL);
         }
   }
}
```

then register this with your DBI instance when youre setting things up:

```java
DBI dbi = new DBI(datasource);
dbi.registerArgumentFactory( new NullArgumentFactory() );
```
