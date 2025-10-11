---
layout: post
comments: false
title:  "Basic DB checklist for optimized performance"
date:   2025-02-11 10:00:00
---

We can't (practically) build applications without databases. Databases are everywhere.
Oftentimes, we run into database issues like slow running queries, high response times, sometimes database crash.

While *understanding and optimizing slow queries queries* and *how databases internally work* is an expertise in itself, but before stating the database is slow or taking huge disk space, you can start by checking the following simple, yet effective things:

➡️ You have optimally normalized the database schema and referenced foreign keys properly.

➡️ You have used column sizes as much required, for example, if `varchar(1)` is ok you don't use `varchar(1024)`; if `short` is ok, you don't use `int`.

➡️You have not written `select * ` in queries, you have at least one `where` clause in your query and they are paginated with `limit` and `offset` as (and where) required. ⭐ (You perhaps would never need `select *` for data tables!)

➡️ You have identified the searchable columns (columns on which we will frequently use `where` clauses), and you have indexed them.

➡️ You have also indexed the "sortable" using `order by` and `group by` columns.

➡️ You have provisioned infra for your database that has main memory 1.5-2x of size of all the indices combined (1.5-2x is a good starting point, throw a bunch of load on your database and understand what's optimal for you).

➡️ You have used Connection Pools.

➡️ You are doing batch inserts wherever you can. ⭐⭐

➡️ (If you run delete statements quite often) You are doing soft deletes, later bulk delete as a periodic housekeeping. ⭐⭐

➡️ Most importantly, your application is not making calls to the database just because it can... You optimize for database calls and hit the database only when you have to, and fetch just as musch data required.

➡️ For tables with huge (meaning, really huge) number of rows... you have used partitioning. [How to do partitioning on Postgres?](https://www.postgresql.org/docs/current/ddl-partitioning.html)


⭐ You would perhaps never require to run a `select` query that fetches all the data from the table; ideally never. If you think you need it, re-analyse your requirements. Sometimes it's important to simplify the problem instead of simplifying the solution.

⭐⭐ Every time we run an `insert` or a `delete` query, right after the `commit` the database re-adjust the indices.
For bulk inserts, the database will re-adjust the indices after the bulk insert, instead of re-adjusting them after every `insert`.
Same for `delete`. Soft delete is done by updating a column like `is_deleted` to `false` and capturing a time stamp. So this is an `update` rather than a `delete`.
Then you can have something as simple as a cron job (that may run during low traffic hours), to periodically bulk delete such soft deleted records at least seven days old.
You get the same advantage like you get during bulk inserts as explained above.

ORMs optimize database hits with bulk operations.

You can do it with raw SQL also, as a software engineer, this is one of the things you optimize for, that shows your maturity.

Another tip, for example in Postgres, use `explain` to understand how the query is executed. This is your first step to identify why a certain query is slow.
