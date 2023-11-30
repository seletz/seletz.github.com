---
layout: post
title: "Jython, Oracle and JDBC"
date: 2012-02-28 08:21
comments: true
categories: jython python howto
---

I recently had the need to access a new table in an existing oracle database
using Jython.  I needed to atomically increase a field in for **one** row.

Here is what I came up with.

<!-- more -->

Notes
-----

- The demo code relies on a existing table 'NXTEST' in oracle.  I used navicat lite
  to create this table.

- The demo code locks a table row using 'select ... for update' to atomically increase
  the SEQ column.  

- The requirement was that I could have a huge number of `name` entries,
  all slightly different, each needing a separate counter.  I did'nt want
  to create a `sequence` for each of them.

Gotchas
-------

- Apparently oracle *needs* upper cased table and column names.

- I could not get oci connections to work.  This seems to be a [bug in 10.2] [1]

- Because of a bug in Jython 2.5.2 wrt `__import__` I could not get
  SQLAlchemy to work.

```python
from __future__ import with_statement

import logging
from com.ziclix.python.sql import zxJDBC

logger = logging.getLogger("nexiles.wt.demo")

def setup_logging(level=logging.DEBUG):
    logging.basicConfig(level=level, format="%(asctime)s [%(levelname)-7s] [line %(lineno)d] %(name)s: %(message)s")

def get_connection(host, port, sid, username, password):
    jdbc_url = "jdbc:oracle:thin:@%(host)s:%(port)s:%(sid)s" % locals()
    logger.debug("get_connection: jdbc_url=%s" % jdbc_url)
    driver = "oracle.jdbc.driver.OracleDriver"
    return zxJDBC.connect(jdbc_url, username, password, driver)

def update_sequence(cursor, name):
    cursor.execute("select NAME, SEQ from NXTEST where name = ? for update", [name])
    result = cursor.fetchall()
    if result:
        logger.debug("UPDATE: %s" % name)
        cursor.execute("update NXTEST set seq = seq + 1 where name = ?", [name])
    else:
        logger.debug("CREATE: %s" % name)
        cursor.execute("insert into NXTEST values (?, ?)", [name, 1])

    cursor.execute("select NAME, SEQ from NXTEST where name = ?", [name])
    result = cursor.fetchall()
    _, seq = result[0]
    logger.debug("NAME: %s => SEQ %03d" % (name, seq))
    return seq

def main():
    username = "****"
    password = "****"

    with get_connection("example.com", 1521, "orc4", username, password) as conn:
        with conn:
            logger.debug("connection: %r" % conn)
            with conn.cursor() as c:
                logger.debug("cursor: %r" % c)

                update_sequence(c, "test-foo-bar")
                update_sequence(c, "test-foo-bar")
                update_sequence(c, "test-bar-bar")
                update_sequence(c, "test-bar-bar")
                update_sequence(c, "flirz-foo-bar")
                update_sequence(c, "flirz-foo-bar")
                update_sequence(c, "flirz-foo-bar")


if __name__ == '__main__':
    setup_logging()
    main()
```

Edit
----

I retested `SQLAlchemy` using the recently [released] [2] `Jython-2.5.3b1`, and
this version indeed fixes the `__import__` bug.

Here's the `homebrew` recipe for it:

```ruby
require 'formula'

class Jython < Formula
  homepage 'http://www.jython.org'
  url "http://downloads.sourceforge.net/project/jython/jython-dev/2.5.3b1/jython_installer-2.5.3b1.jar",
    :using => :nounzip
  sha1 'bcfc024a93289b2f99bf000fb7666a48fe3d32da'

  def install
    system "java", "-jar", "jython_installer-2.5.3b1.jar", "-s", "-d", libexec
    bin.install_symlink libexec+'bin/jython'
  end
end
```

FWIW, this shows how to connect to Oracle using `SQLAlchemy` and Jython.
Thoe code below does **not** do the same thing as the one above (as in
atomic increases of the SEQ column), however.

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from sqlalchemy import Sequence
from sqlalchemy import Table, Column, String, Integer, MetaData

from sqlalchemy.orm import mapper

url = "oracle+zxjdbc://user:pass@example.com:1521/orc4"

engine = create_engine(url, echo=True)

metadata = MetaData()
fooversions_table = Table('FooVersions', metadata,
        Column('id', Integer, Sequence('foo_version_id_seq'), primary_key=True),
        Column('name', String(50)),
        Column('seq', Integer)
        )

metadata.create_all(engine)

class FooVersions(object):
    def __init__(self, name, seq):
        self.name = name
        self.seq = seq

    def __repr__(self):
        return "<FooVersions('%s','%s')>" % (self.name, self.seq)

m = mapper(FooVersions, fooversions_table)

# Create Session class and bind it to the database
Session = sessionmaker(bind=engine)
session = Session()

session.add( FooVersions("context-foo-abc", 1) )
session.add( FooVersions("context-foo-def", 1) )
session.add( FooVersions("context-foo-ghi", 1) )

session.commit()
```

Still, I see the following advantages using `SQLAlchemy` (other than having
a pretty darn nifty ORM):

- Class Mappers.  I like these.

- `SQLAlchemy` hides all the magic Oracle stuff (Sequences, upper cased table
  names)

- Minor but handy: Optionally `SQLAlchemy` logs every SQL statement.

[1]:  https://forums.oracle.com/forums/thread.jspa?threadID=1080064  "bug in 10.2"
[2]:  http://downloads.sourceforge.net/project/jython/jython-dev/2.5.3b1/jython_installer-2.5.3b1.jar "released"
