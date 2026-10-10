# SQLAlchemy
A Python database toolkit as well as an Object Relational Mapper (ORM).
It consists of two parts:
- **Core** - the database toolkit: connections, and building SQL statements from Python objects instead of writing SQL strings.
- **ORM (Object Relational Mapper)** - used to define DB tables as Python classes, and work with the DB in the OOP (Object-Oriented Programming) paradigm: individual rows are objects, tables are classes. Built on top of SQLAlchemy Core.

SQLAlchemy does not care about the underlying DB - it can work with SQLite, MySQL, PostgreSQL, etc.

## Install
Install with `uv add sqlalchemy`. To also install the extra package needed for async mode, we use `uv add "sqlalchemy[asyncio]"` (the quotes stop some shells, like zsh, from treating `[...]` as a file pattern).

SQLAlchemy doesn't talk to the database itself; it needs a **driver** for that database. For PostgreSQL: `uv add asyncpg` for async, `uv add "psycopg[binary]"` for sync. SQLite needs nothing extra: its driver is built into Python.

The driver is named in the URL: `postgresql+psycopg://...` or `postgresql+asyncpg://...`.

## Run basic queries
To run basic queries, we can just create an engine, connect to it, and execute SQL:
```python
from sqlalchemy import create_engine, text

engine = create_engine(
    "postgresql+psycopg://<user>:<password>@<host>:<port>/<database>", echo=True
)  # echo=True logs every SQL statement

with engine.connect() as conn:  # the connection is returned to the pool at the end of the block
    conn.execute(text("CREATE TABLE IF NOT EXISTS people (name TEXT, age INTEGER)"))
    conn.execute(text("INSERT INTO people VALUES ('Messi', 39), ('Ronaldo', 41)"))
    conn.commit()  # save the table and the rows; without it, they are rolled back

    result = conn.execute(text("SELECT * FROM people"))
    for row in result:  # each row behaves like a tuple, and columns are also attributes
        print(row.name, row.age)
```
One statement per `execute`: several statements in one string give back only one result and make errors harder to trace.

We can also do it with a `Session` (part of the ORM, but it can run plain SQL too):
```python
from sqlalchemy import create_engine, text
from sqlalchemy.orm import Session

engine = create_engine(
    "postgresql+psycopg://<user>:<password>@<host>:<port>/<database>", echo=True
)

with Session(engine) as session:  # the session is closed at the end of the block
    session.execute(text("<some query>"))
    session.commit()
```

## SQLAlchemy Core
Core can describe a table with the `Table` type, and then build statements from Python, without writing SQL strings:
```python
from sqlalchemy import Column, Integer, MetaData, String, Table, create_engine

engine = create_engine(
    "postgresql+psycopg://<user>:<password>@<host>:<port>/<database>"
)

meta = MetaData()  # a collection of table definitions

table = Table(
    "table_name",
    meta,
    Column("id", Integer, primary_key=True),
    Column("column1", String, nullable=False),
    Column("column2", Integer),
)

meta.create_all(engine)  # create every table in meta that doesn't exist yet
```
A table's columns are reached through `.c`: `table.c.column1`.

Insert:
```python
with engine.connect() as conn:
    insert_statement = table.insert().values(column1="value1", column2=2)
    conn.execute(insert_statement)
    conn.commit()
```

Select:
```python
from sqlalchemy import select

with engine.connect() as conn:
    select_statement = select(table).where(table.c.column2 < 10)
    result = conn.execute(select_statement)
    for row in result.fetchall():
        print(row)
```

Update:
```python
with engine.connect() as conn:
    update_statement = table.update().where(table.c.column1 == "value1").values(column2=3)
    conn.execute(update_statement)
    conn.commit()
```

Delete:
```python
with engine.connect() as conn:
    delete_statement = table.delete().where(table.c.column1 == "value1")
    conn.execute(delete_statement)
    conn.commit()
```

### Relations and joins
A second table can point to the first with a foreign key, and a join combines them:
```python
from sqlalchemy import Column, Float, ForeignKey, Integer, MetaData, String, Table, select

meta = MetaData()

table1 = Table(
    "table1",
    meta,
    Column("id", Integer, primary_key=True),
    Column("column1", String),
    Column("column2", Integer),
)

table2 = Table(
    "table2",
    meta,
    Column("id", Integer, primary_key=True),
    Column("column1", String),
    Column("column2", Float),
    Column("column3", Integer, ForeignKey("table1.id")),  # must be an existing table1.id
)

meta.create_all(engine)

# insert some data...

join = table1.join(table2, table1.c.id == table2.c.column3)  # inner join; table1.outerjoin(...) for a LEFT JOIN
select_statement = select(table1.c.column1, table2.c.column2).select_from(join)

with engine.connect() as conn:
    result = conn.execute(select_statement)
    for row in result.fetchall():
        print(row)
```

### Grouping
With the same `table1`:
```python
from sqlalchemy import func, select

group_by_statement = (
    select(table1.c.column1, func.sum(table1.c.column2))  # func.<name> calls a SQL function: SUM, COUNT, AVG, ...
    .group_by(table1.c.column1)
    .having(func.sum(table1.c.column2) > 10)              # optional: keep only groups whose sum is over 10
)

with engine.connect() as conn:
    result = conn.execute(group_by_statement)
    for row in result.fetchall():
        print(row)
```
