---
title: "Default Extensions in PostgreSQL"
tags: postgresql
date: 2020-06-10
---

While RTFMing – I’m trying to learn more about PostgreSQL by working through
*Learning PostgreSQL 11* – I found this:

> Every database created by PostgreSQL inherits from a template database.
> Normally this is template1.  This is very useful, as the database cluster admin
> now can create extensions in that template, which all then be available from all
> newly created databases.

Why is this useful?  The `CREATE EXTENSION` privilege in PostgreSQL cannot be
granted to non-superusers.  In a cluster / K8S setup, typically the Postgres
superuser is hidden and never used.  So we have a catch-22 situation.

We solved this by customising docker containers and / or supplying init scripts
which are run on the first time.  This is OK – but this only works when the
container starts the first time.

Instead, we could simply alter `template1` in the init script like so:

```sql
\c template1;

CREATE EXTENSION ltree;
CREATE EXTENSION postgis;
```

Then, every database created will have these extensions available.

