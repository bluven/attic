---
title: "The trouble with soft delete"
date: 2018-01-15T22:46:05+08:00
draft: false
---

**Soft delete** is a commonly-used pattern amongst database-driven business applications. In my experience, however, it usually ends up causing more harm than good. Here’s a few reasons why it can fail in bigger applications, and some less-painful alternatives to consider.

## Tomato, Tomato

I’ve seen a few different implementations of this pattern in action. First is the standard deleted flag to indicate an item should be ignored:

```
SELECT * FROM Product WHERE IsDeleted = 0
```

Another style uses meaningful status codes:

```
SELECT * FROM Task WHERE Status = 'Pending'
```

You can even give an item a fixed lifetime that starts and ends at a specific time (it might not have started yet):

```
SELECT * FROM Policy WHERE GETDATE() BETWEEN StartDate AND EndDate
```

All of these styles are all flavours of the same concept: instead of pulling dead or infrequently-used items out of active set, you simply mark them and change queries to step over the corpses at runtime.

This is a trade-off: soft delete columns are easy to implement, but incur a cost to query complexity and database performance later down the track.

## Complexity

To prevent mixing active and inactive data in results, all queries must be made aware of the soft delete columns so they can explicitly exclude them. It’s like a tax; a mandatory WHERE clause to ensure you don’t return any deleted rows.

This extra WHERE clause is similar to checking return codes in programming languages that don’t throw exceptions (like C). It’s very simple to do, but if you forget to do it in even one place, bugs can creep in very fast. And it is background noise that detracts away from the real intention of the query.

## Performance

At first glance you might think evaluating soft delete columns in every query would have a noticeable impact on performance.

However, I’ve found that most RDBMSs are actually pretty good at recognizing soft delete columns (probably because they are so commonly used) and does a good job at optimizing queries that use them. In practice, filtering inactive rows doesn’t cost too much in itself.

Instead, the performance hit comes simply from the volume of data that builds up when you don’t bother clearing old rows. For example, we have a table in a system at work that records an organisations day-to-day tasks: pending, planned, and completed. It has around five million rows in total, but of that, only a very small percentage (2%) are still active and interesting to the application. The rest are all historical; rarely used and kept only to maintain foreign key integrity and for reporting purposes.

Interestingly, the biggest problem we have with this table is not slow read performance but writes. Due to its high use, we index the table heavily to improve query performance. But with the number of rows in the table, it takes so long to update these indexes that the application frequently times out waiting for DML commands to finish.

This table is becoming an increasing concern for us — it represents a major portion of the application, and with around a million new rows being added each year, the performance issues are only going to get worse.

## Back to the original problem

The trouble with implementing soft delete via a column is that it simply doesn’t scale well for queries targeting multiple tables — we need a different strategy for larger data models.

Let’s take a step back and examine the reasons why you might want to implement soft deletes in a database. If you think about it, there really are only four categories:
 
    1. To provide an ‘undelete’ feature.
    2. Auditing.
    3. For soft create.
    4. To keep historical items.

Let’s look at each of these and explore what other options are available.

### Soft delete to enable undo

Human error is inevitable, so it’s common practice to give users the ability to bring something back if they delete it by accident. But this functionality can be tricky to implement in a RDBMS, so first you need to ask an important question — do you really need it?

There are two styles I have encountered that are achieved via soft delete:

    1. There is an undelete feature available somewhere in the UI, or
    2. Undelete requires running commands directly against the database.

If there is an undo delete button available somewhere for users, then it is an important use case that needs to be factored into your code.

But if you’re just putting soft delete columns on out of habit, and undelete still requires a developer or DBA to run a command against the database to toggle the flags back, then this is a maintenance scenario, not a use case. Implementing it will take time and add significant complexity to your data model, and add very little benefit for end users, so why bother? It’s a pretty clear YAGNI violation — in the rare case you really do need to restore deleted rows, you can just get them from the previous night’s backup.

Otherwise, if there really is a requirement in your application for users to be able to undo deletes, there is already a well-known pattern specifically designed to take care of all your undo-related scenarios.

### The memento pattern

Soft delete only supports undoing deletes, but the memento pattern provides a standard means of handling all undo scenarios your application might require.

It works by taking a snapshot of an item just before a change is made, and putting it aside in a separate store, in case a user wants to restore or rollback later. For example, in a job board application, you might have two tables: one transactional for live jobs, and an undo log that stores snapshots of jobs at previous points in time:

If you want to restore one, you simply deserialize it and insert it back in. This is much cleaner than messing up your transactional tables with ghost items, and lets you handle all undo operations together using the same pattern.

### Soft delete for auditing

Another common practice I have seen is using soft delete as a means of auditing: to keep a record of when an item was deleted, and who deleted it. Usually additional columns are added to store this information:

As with undo, you should ask a couple of questions before you implement soft delete for auditing reasons:

* Is there a requirement to log when an item is deleted?
* Is there a requirement to log any other significant application events?

In the past I have seen developers (myself included) automatically adding delete auditing columns like this as a convention, without questioning why it’s needed (aka cargo cult programming).

Deleting an object is only one event we might be interested in logging. If a rogue user performs some malicious acts in your application, updates could be just as destructive so we should know about those too.

```
-- Log all DELETE operations on the Product table
CREATE TRIGGER tg_Product_Delete ON Product AFTER DELETE
AS 
	INSERT INTO [Log]
	(
		[Timestamp],
		[Table],
		Command,
		ID
	)
	SELECT
		GETDATE(),
		'Product',
		'DELETE',
		ProductID
	FROM
		Deleted

CREATE TRIGGER tg_Product_Update ON Product AFTER UPDATE
AS
-- ...etc

```

### Contextual logging

If something goes wrong in the application and we want to retrace the series of steps that led to it, CREATES, UPDATES and DELETES by themselves don’t really explain much of what the user was trying to achieve. Getting useful audit logs from DML statements alone is like trying to figure out what you did last night from just your credit card bill.

It would be more useful if the logs were expressed in the context of the use case, not just the database commands that resulted from it. For example, if you were tracking down a bug, would you rather read this:

```
[09:30:24] DELETE ProductCategory 142 13 dingwallr
```

… or this?

```
[09:30:24] Product 'iPhone 3GS' (#142) was removed from the catalog by user dingwallr. Categories: 'Smart Phones' (#13), 'Apple' (#15).
```

Logging at the row level simply cannot provide enough context to give a true picture of what the user is doing. Instead it should be done at a higher level where you know the full use-case (e.g. application services), and the logs should be kept out of the transactional database so they can be managed separately (e.g. rolling files for each month).

### Soft create

The soft delete pattern can also be extended for item activation — instead of simply creating an item as part of the active set, you create it in an inactive state and flick a switch or set a date for when it should become active.

For example:

```
-- Get employees who haven't started work yet
SELECT * FROM Employee WHERE GETDATE() &lt; StartDate
```

This pattern is most commonly seen in publishing systems like blogs and CMSs, but I’ve also seen it used as an important part of an ERP system for scheduling changes to policy before it comes into effect.

### Soft delete to retain historical items

Many database-backed business applications are required to keep track of old items for historical purposes — so users can go back and see what the state of the business was six months ago, for example.

(Alternatively, historical data is kept because the developer can’t figure out how to delete something without breaking foreign key constraints, but this really amounts to the same thing.)

We need to keep this data somewhere, but it’s no longer immediately interesting to the application because either:

* The item explicitly entered a dormant state – e.g. an expired eBay listing or     deactivated Windows account, or
* The item was implicitly dropped from the active set – e.g. my Google Calendar appointments from last week that I will probably never look at again

I haven’t included deleted items here because there are very few cases I can think of in business applications where data is simply deleted (unless it was entered in error) — usually it is just transformed from one state to another. Udi Dahan explains this well in his article [Don’t Delete — Just’ Don’t](http://udidahan.com/2009/09/01/dont-delete-just-dont/):

> Orders aren’t deleted – they’re cancelled. There may also be fees incurred if the order is canceled too late.

> Employees aren’t deleted – they’re fired (or possibly retired). A compensation package often needs to be handled.

> Jobs aren’t deleted – they’re filled (or their requisition is revoked).

Unless your application is pure CRUD (e.g. data grids), these states most likely represent totally different use cases. For example, in a timesheet application, complete tasks may be used for invoicing purposes, while incomplete tasks comprise your todo list.

### Different use cases, different query needs

Each use case has different query requirements, as the information the application is interested in depends on the context. To achieve optimal performance, the database should reflect this — instead of lumping differently-used sets of items together in one huge table with a flag or status code as a discriminator, consider splitting them up into separate tables.

For example, in our job board application, we might store open, expired and filled listings using a table-per-class strategy:

Physically separating job listings by state allows us to optimize them for different use cases — focusing on write performance for active items, and read performance for past ones, with different columns, indexes and levels of (de)normalization for each.

### Isn’t this all overkill?

Probably. If you’re already using soft delete and haven’t had any problems then you don’t need to worry — soft delete was a sensible trade-off for your application that hasn’t caused any serious issues so far.

But if you’re anticipating growth, or already encountering scalability problems as dead bodies pile up in your database, you might like to look at alternatives that better satisfy your application’s requirements.

The truth is soft delete is a poor solution for most of the problems it promises to solve. Instead, focus on what you’re actually trying to achieve. Keep everything simple and follow these guidelines:


* Primary transactional tables should only contain data that is valid and active right now.
* Do you really need to be able to undo deletes? If so, there are dedicated patterns to handle this.
* Audit logging at the row level sucks. Do it higher up where you know the full story.
* If a row doesn’t apply yet, put it in a queue until it does.
* Physically separate items in different states based on their query usage.

Above all, make sure you’re not gold plating tables with soft delete simply out of habit!

