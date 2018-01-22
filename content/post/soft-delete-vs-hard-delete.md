---
title: "Soft Delete vs Hard Delete"
date: 2018-01-15T22:45:05+08:00
draft: false
---

## About

* I have been part of quite a few projects where soft deleting was the trend.
* Over the last 3-4 years, I have had a chance to analyze the pros/cons of this approach in real world projects.
* This post attempts to document my observations on this trend.

## Soft Delete

* Involves setting a column like deleted_flag when the row has to be ‘deleted’.
* All queries involving the active flow of the application will ignore such entries with the condition ‘deleted_flag = 0’

## Hard Delete

* Involves directly deleting the data row from the table.
* Many a times, the data (or action) is copyed onto an audit table to assist in possible debugging in the future.
* This copying can be done with one of the following options : -- Listener code in the application (observer pattern) -- Database Trigger -- Combine the copy + delete operations in a stored procedure.

## Comparison Factors :

### Ease of Setup

* Soft Delete is easier to implement since it merely involves updating a column while hard delete would also involve copying the data to be deleted to an audit table.
* **Advantage : Soft Delete**

### Debugging

* Soft Delete makes it easy to debug data issues since the deleted_flag
* But debugging via the Audit table is also easily possible. So its a tie.
* **Advantage : NA**

### Restoring data

* It is extremely easy to restore data 'deleted' via soft delete since it just involved unsetting the deleted_flag.
* However note that restoring data is an extremely rare occurance.
* **Advantage : Soft Delete**

### Querying for active data

* By experience, I can state that many issues have come up when the developer has forgotten to add ‘delete_flag = 0’ condition in the select queries due to which issues came about.
* If you are using an ORM like Doctrine with the ‘soft delete’ plugin enabled, then this will not be an issue since the ORM takes care of adding this check.
* **Advantage : Hard Delete**

### View Simplicity

* Having all the data in the tables as active data relates to view simplicity (WYSIWYG - What you see is what you get)
* In Hard delete, all ‘deleted’ data will only be present in the audit table while the rest of the tables in the system will have ‘active’ data. So the separation of concerns exists for hard delete.
* **Advantage : Hard Delete**

### Performance of operation

* Update is a bit faster than delete (microseconds)
* So soft delete should technically be faster than hard delete (which also has the audit table insert to consider).
* **Advantage : Soft Delete**

### Application Performance

* **Speed**

    * To support soft deletes, ALL select queries need to have a condition ‘WHERE delete_flag = 0’.
    * In situations where JOINs are involved there will be multiple such conditions.
    * Select queries with lesser conditions are faster than those with conditions.
    * **Advantage : Hard Delete**

* **Size**
    * To support faster soft deletes, we need to have an index for every        delete_flag in EVERY table
    * Additionally the table size keeps increasing since the table has ‘soft deleted’ data + active data.
    * Queries can get slower as table size increases.
    * **Advantage : Hard Delete**

### Database features compatibility

* **Unique Index**

    * Unique index ensures data integrity by preventing multiple occurrences of a row at the database level.
    * Having soft delete prevents usage of Unique index.
        * Eg :
            * Field A & Field B & deleted_flag have a composite unique index.
            * if the rows having the data A1 & B1 are ‘soft deleted’ then the unique index will be for values A1-B1-1 (ie deleted_flag).
            * If a new entry A1-B1 is added to the table then it cannot be soft deleted again due to the unique index.
        * Additionally we cannot update the old soft deleted entry of A1-B1 since it would mean rewriting some data which results in loss of recorded data (eg : update date time or some other deleted_by column if it exists)
        * **Advantage : Hard Delete**
        
* **Cascading**

   * For soft delete, we cannot make use of ‘ON DELETE’ cascading.
   * The alternative is to create an 'UPDATE' trigger which keeps track of deleted_flag.
   * **Advantage : Hard Delete**


## Recommendation

Hard delete approach looks to have more advantages than soft delete which is why it is something I recommend over soft delete.


> *原地址：http://rohithegde.github.io/soft-vs-hard-delete/*
