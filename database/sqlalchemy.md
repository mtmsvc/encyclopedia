# SQLAlchemy
A Python database toolkit as well as an Object Relational Mapper (ORM).
It consists of two parts:
- **Core** - database toolkit for Python
- **ORM (Object Relational Mapper)** - used to define DB tables as Python classes, and work with DB. We can use OOP (Object-Oriented Programming) paradigm to interact with DBs (treat individual rows as objects, tables as classes). Built on top of SQLAlchemy Core.

SQLAlchemy does not care about the underlying DB - it can work with SQLite, MySQL, PostgreSQL, etc.

## Install
Install with `uv add sqlalchemy`. In order to also install extra packages for async mode, we use `uv add sqlalchemy[asyncio]`.

To talk to PostgreSQL, we should also install the driver: `uv add asyncpg` for async, `uv add psycopg[binary]` for sync.

## Run basic queries
To run basic queries, we can just create an engine, connect to it, and execute commands:
```python
from sqlalchemy import create_engine, text

engine = create_engine(
    "postgresql+psycopg://<user>:<password>@<host>:<port>/<database>", echo=True
)  # echo=True logs every SQL statement

with engine.connect() as conn:
    conn.execute(text("CREATE TABLE IF NOT EXISTS table (field1 TYPE1, field2, TYPE2"))
    conn.execute(text("INSERT INTO people VALUES (..., ...), (..., ...)"))
    conn.commit()  # save the table and the rows

    result = conn.execute(text("SELECT * FROM table"))
    for row in result:  # each row behaves like a tuple
        print(row.field1, row.field2)
```

We can also do it with `Session`:
```python
from sqlalchemy.orm import Session
from sqlalchemy import create_engine, text

engine = create_engine(
    "postgresql+psycopg://<user>:<password>@<host>:<port>/<database>", echo=True
)  # echo=True logs every SQL statement

session = Session(engine)

session.execut(text("<some query>"))

session.commit()
```

## SQLAclhemy Core
There is a type `Table` in SQLAlchemy, that we can use to interact with DB directly from code, without writing queries:
```python
from sqlalchemy import create_engine, MetaData, Table, Column, Integer, String


engine = create_engine(
    "postgresql+psycopg://<user>:<password>@<host>:<port>/<database>"
)

meta = MetaData()

table = Table(
    "table_name",
    meta,
    Column('id', Integer, primary_key=True),
    Column('column1', String, nullable=False),
    Column('column2', Integer)
)

meta.create_all(engine)

conn = engine.connect()
```

We can insert stuff:
```python
insert_statement = table.insert().values(column1="value1", column2=2)
result = conn.execute(insert_statement)
conn.commit()
```

Select:
```python
select_statement = table.select().where(table.column2<10)
result = conn.exectue(select)
for row in result.fetchall():
    print(row)
```

Update:
```python
update_statement = table.update().where(table.column1=="value1").values(column2=3)
result = conn.execute(update_statement)
conn.commit()
```

Delete:
```python
delete_statement = table.delete().where(table.column1=="value1")
result = conn.execute(delete_statement)
conn.commit()
```

We can also create another table and use relations with JOIN statements:
```python
from sqlalchemy import create_engine, MetaData, Table, Column, Integer, String, ForeignKey, Float


engine = create_engine(
    "postgresql+psycopg://<user>:<password>@<host>:<port>/<database>"
)

meta = MetaData()

table1 = Table(
    "table1",
    meta,
    Column('id', Integer, primary_key=True),
    Column('column1', String),
    Column('column2', Integer)
)

table2 = Table(
    "table2",
    meta,
    Column('id', Integer, primary_key=True),
    Column('column1', String),
    Column('column2', Float)
    Column('column3', Integer, ForeignKey('table1.id'))
)

meta.create_all(engine)

conn = engine.connect()

# insert some data...

join_statement = table1.join(table2, table1.id==table2.column3) # we can also use outerjoin, etc.
select_statement = table1.select().with_only_columns(table1.column1, table2.column2).select_from(join_statement)
result = conn.exectue(select_statement)
for row in result.fetchall():
    print(row)
```

We can also group:
```python
from sqlalchemy import create_engine, MetaData, Table, Column, Integer, String, ForeignKey, Float, func


engine = create_engine(
    "postgresql+psycopg://<user>:<password>@<host>:<port>/<database>"
)

meta = MetaData()

table1 = Table(
    "table1",
    meta,
    Column('id', Integer, primary_key=True),
    Column('column1', String),
    Column('column2', Integer)
)

table2 = Table(
    "table2",
    meta,
    Column('id', Integer, primary_key=True),
    Column('column1', String),
    Column('column2', Float)
    Column('column3', Integer, ForeignKey('table1.id'))
)

meta.create_all(engine)

conn = engine.connect()

# insert some data...

group_by_statement = table1.select().with_only_columns(table1.column1, func.sum(table1.column2)).group_by(table1.column1)

# we could add filtering:
# group_by_statement = table1.select().with_only_columns(table1.column1, func.sum(table1.column2)).group_by(table1.column1).having(func.sum(table1.column2)>10)
result = conn.exectue(group_by_statement)
for row in result.fetchall():
    print(row)
```
