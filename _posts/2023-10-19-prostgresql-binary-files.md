---
layout:    post
title:    "Read binary files into a bytea column in PostgreSQL"
date:     2023-10-09
comments: true
tags: postgresql 
---

I recently needed to test some SQL queries involving a table which contains
column of  type bytea with a not null constraint.  The table was empty and I
wanted to add some test data.

It turns out that PostgreSQL has a `pg_read_binary_file()` function which can
be used to do exactly that.  The function – as the name suggests – read a file
on the PostgreSQL server filesystem and returns binary data.

For a PG user to be allowed to `EXECUTE` such functions, you need to grant
`EXECUTE` for the function, specifying the full function signature:


```sql
GRANT EXECUTE ON FUNCTION pg_read_binary_file(text,bigint,bigint,boolean) TO revisor;
GRANT EXECUTE ON FUNCTION pg_read_binary_file(text,bigint,bigint) TO revisor; 
GRANT EXECUTE ON FUNCTION pg_read_binary_file(text) TO revisor;
```

Then do:

```sql
insert into mytable values (  
    1, 'foo.pdf', pg_read_binary_file('foo.pdf')
)
```



> When using postgres in a docker container -- be sure that the file is actually visible in the container and use the path inside the container.
> Easiest way is to add the file to the `data` directory -- you can use symlinks if needed.
{: .prompt-tip }

> File **MUST** be in the data directory or you need to additionally grant `pg_read_server_files` role
{: .prompt-warning }


See also [PostgreSQL Docs](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADMIN-GENFILE).
