+++
author = "hannibal"
categories = ["programming"]
date = "2014-05-31"
tags = ["schemaverse", "sql"]
title = "Five reasons why a tester should learn SQL"
url = "/2014/05/31/five-reasons-why-a-tester-should-learn-sql/"

+++

Hello Folks.

So last I was writing about why a tester should learn Javascript. Today I would like to write about why a tester should learn SQL.

There, I said it. I know many people, especially testers, don't like SQL. They view it as a monster best be avoided. Something only Database people will know. Something which is so scary and ugly, nobody really wants it.

But I will give you a couple of good reasons why you shouldn't be afraid of SQL. And why you should welcome it as your best friend and partner in crime.

Let's go.

<!--more-->

**Reason Number One: Data gathering**

This one is obvious and clear as the sun. You can use simple queries to mine for data. To look up changes and compare time intervals between insertion and update events. You can monitor certain tables so that when they are updated, you'll get a red flag.

**Reason Number Two: Test Run Statistics**

This one is from my friend Adrian. Basically if you would like to know more about what's happening with your tests when they run, for example:

  * Run time
  * Frequency of Test outcome
  * Failure rate
  * Environment details -> Execution slave

One interesting way to achieve this is, to have the test report running and outcomes into a little mysql database and than create queries of certain types like, show me the last run of every test called xyz and show me the environmental details and the run time. With this closely monitored you could find out that Slave #345 is sluggish because each time the test ran on it, it took more then 10 minutes where as the others only took 5-6.

**Reason Number Three: Data Manipulation**

So after monitoring comes naturally the edit. Understanding how databases work and knowing a few queries here and there can help you manipulating your data in a way that it will be easier to test data dependant scenarios.

Instead of making a new entry you could edit what you already have.

For example:

You have a customer and you want to test the system's ability to handle people on your site who are suspended from access. But the ability to suspend is not yet working. Will that stop you from testing this feature? Will you put this feature into blocked because: "Ohh, we can't yet suspend a player so we need to wait until that's done."

No. You don't wait. You dig into your database, find the necessary record, change it's state to the desired state; even if it has a foreign key which needs to be updated or that status doesn't exist yet, in which case you ADD it to the list of states yourself. You don't let yourself be stopped just because something is not done yet. You move on by being clever.

**Reason Number Four: Understanding Data migration issues**

A big issue these days if you go into a project where you have a previous version of what you are currently building will be migrating over old data into the new database scheme. Testing such a thing can be a pain in the butt. But it will be a LOT easier if you understand the changes. If you know what changed, how and why, you can manipulate your data in order to fit the new scheme. Or if you need to test it you won't be afraid of running some stored procedures in a dummy database with old data and than run a few queries to see what broke and what didn't.

Do you have a foreign key violation somewhere when you migrated over and have no idea what do to? Time to learn some SQL so that you don't have to run to somebody every time you encounter it. Even if you can't fix it, the database engineers or the devs who will fix the bug will be very happy that you provided as much information as possible in your report.

**Reason Number Five: Security**

SQL Injection is still at large. Even with these days frameworks doing full escapes it can't hurt to test a couple just to be on the safe side. And writing a clever script that mines for accessible tables here and there is an essential skill in a security tester's repertoire.

**Reason Bonus: Performance, HibernateQL, Information**

Lastly a bit of a bonus are these three.

**Performance**

Suppose you are running a web application. You access a large list and you notice that it's sort of sluggish. You first blame it on the network so you test it locally. It seems to be better now so you move on but it leaves a little voice in your head so you can't abandon it. You go back and try locally again with a bit more data in your database.

You notice it's a little bit better but it's still somehow sluggish. Suddenly you get a hunch and turn on SQL logging in your tomcat instance. You click the link again and wait with eyes wide open on what happens next. It's your worst nightmare.

The SQL queries the whole database with a simple select and than filters the data on the front-end side. Which is a really really dumb thing to do. So you file a bug for which the title is something similar to this: "Customer Transaction History query doesn't apply WHERE | INNER SELECT | PAGING on SQL rather it filters on the application layer."

**HibernateQL**

This is the SQL language of hibernate which is the underlying technology in many java web frameworks these days. It uses it's own thing called <a href="http://docs.jboss.org/hibernate/orm/3.3/reference/en/html/queryhql.html" target="_blank">HQL</a>. Main difference, as this page already says, is that it's a full blown Object Oriented query language which understand inheritance and polymorphism which is very exciting.

**Information**

Last but not least I mentioned this one earlier. You can provide more information in your bug reports as in what data you used, where was it happening, what was the last update date, which environment and what query was executed ( if you have query debugging turned on ). Whoever reads that bug report will find it very helpful that you provided enough information to reproduce it anywhere.

Because many times the culprit for a bug is the underlying data.

**Reading and Practising resources**

Here is a very awesome picture of how to understand JOINS which is everybody's fear.

<div style="width: 976px" class="wp-caption aligncenter">
  <img src="http://www.codeproject.com/KB/database/Visual_SQL_Joins/Visual_SQL_JOINS_orig.jpg" alt="SQL Join Cheat Sheet" width="966" height="760" />

  <p class="wp-caption-text">
    SQL Join Cheat Sheet
  </p>
</div>

And a post on <a href="http://blog.codinghorror.com/a-visual-explanation-of-sql-joins/" target="_blank">Coding Horror</a> which is essentially the same but I like how Jeff writes.

Also if you would like to practice writing SQL scripts and no longer be afraid of them all the rest of your life go to this site => <a href="http://sqlzoo.net/" target="_blank">SQLZoo</a>. It's an interactive way of trying out your SQL skills and testing them on very clever database structures.

But if you, like me, love to learn PLAYING than THIS is the place for you => <a href="https://schemaverse.com/" target="_blank">The Schemaverse</a>. It's a SQL based space shooting strategy game awesomeness! Have FUN learning SQL.

Thanks for reading!

Have a nice day|night!

Gergely.