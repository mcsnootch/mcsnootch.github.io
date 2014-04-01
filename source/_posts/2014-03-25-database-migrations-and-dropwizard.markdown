---
layout: post
title: "Database schema migrations and Dropwizard"
date: 2014-03-25 22:10:18 +0000
comments: true
categories: 
---

One of the problems with writing applications backed by a datastore is that the metadata describing the datastore (DDL in the SQL world) needs to be applied to the datastore in step with the code which accesses that store. Previous versions of the application code may work well with a more advanced version of the schema, but a newer version of your code almost certainly won't work well with an old version of your database.  

<!-- more -->

The noSQL crowd may scoff at these concerns, merrily migrating their data formats as they go[^1], but those of us slaving away with good old relational tech need some way to move our schemas on with the code. To achieve this outside of the ivory tower, we've tried the following approaches: 

   * Running migrations ourselves (horrible, involves very fallible people, highly prone to result in datastores out of sync with application code, slow)
   * Sending sql scripts to DBAs to run in QA/production/wherever (horrible, involves slightly less fallible people, slow)
* Bundling scripts with our releases, getting a DBA to check it out and run in QA/production/wherever (horrible, involves people, again, only slightly less prone to result in a database out of step with application code)
* Writing a platform-specific migration tool which _still_ needs a DBA to press a button (horrible, expensive, buggy, people etc. etc.)
* Using a general purpose migration tool which takes migrations checked into the code, and having DBAs run it in QA/production/wherever (people, bloody people!)

These are all objectively horrible solutions to the problem that our application code isn't taking the responsibility to ensure that it's datastore is compatible with it. So, we've finally settled on checking our migrations alongside our application code and having our applications migrate their own datastore.

I've really been enjoying developing applications with Dropwizard, and amongst it's many lovely features are support for migrations using Liquibase. Unfortunately, it's default migration commands don't run as the server starts up and need to be explicitly called outside of application startup. We could try chucking the migration command into our deployment scripts, but this feels inelegant, or we could skip using Dropwizard's liquibase support and roll our own, parsing or duplicating our application's database configuration ourselves. If you're lazy like me, though, and appreciate the idea of someone else loading config, providing a conventional location for the migration code and all that jazz, here's how to borrow Dropwizard's migration command and reuse it for your own purpose:

      @Override
      public void run(YourServiceConfig config, Environment environment) throws Exception {
         ManagedLiquibase liquibase = new ManagedLiquibase(config.getDatabaseConfiguration());
         liquibase.update("");

         // other setup and wiring here
      }

and that's all there is to it. Starting your Dropwizard application should now migrate your schema to to application appropriate version, without your having to lift a finger.

[^1]: provided, of course, that they've remembered to version the data they've stuffed into the schemaless sump created for themselves, and can work out how to get that customer object created three years ago as version one up to today's version one bazillion without their tests imploding and their QA suffering an aneurysm
