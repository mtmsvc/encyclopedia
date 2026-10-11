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

## SQLAlchemy ORM
The ORM maps tables to Python classes: a class is a table, an attribute is a column, an object is a row. We work with objects, and the **session** works out the SQL for us.

The example: people, and the things they own.

### 1. The base class
```python
from sqlalchemy import ForeignKey, String, create_engine, delete, func, select, update
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column, relationship

engine = create_engine("postgresql+psycopg://blasto:blasto@localhost:5432/blasto")


class Base(DeclarativeBase):
    pass
```
Every table class inherits from `Base`. It collects all table definitions in `Base.metadata` - the same kind of `MetaData` as in Core.

### 2. Tables as classes
```python
class Person(Base):
    __tablename__ = "people"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String)
    age: Mapped[int | None]

    things: Mapped[list[Thing]] = relationship(back_populates="person")


class Thing(Base):
    __tablename__ = "things"

    id: Mapped[int] = mapped_column(primary_key=True)
    description: Mapped[str]
    value: Mapped[float | None]
    owner: Mapped[int | None] = mapped_column(ForeignKey("people.id"))

    person: Mapped[Person] = relationship(back_populates="things")
```
- `__tablename__` - the table's name in the database.
- `Mapped[...]` - the column's Python type. SQLAlchemy derives the database type from it: `int` → `INTEGER`, `str` → `VARCHAR`, `float` → `FLOAT`.
- `| None` - the column may be empty (`NULL`). Without it, the column is required (`NOT NULL`).
- `mapped_column(...)` - database details the type alone doesn't say: `primary_key=True`, a specific type like `String`, a default, a foreign key. With nothing to add, it can be left out, like `age` or `description`.
- `ForeignKey("people.id")` - a real column: `owner` must contain an existing `people.id`.
- `relationship(...)` - **not** a column; it exists only in Python. `person.things` is the list of that person's `Thing` objects, and `thing.person` is the `Person` object - no looking up IDs by hand.
- `back_populates` - links the two relationships, so both sides stay in sync.
- `Mapped[list[Thing]]` uses `Thing` before it is defined. That works because Python 3.14 evaluates annotations lazily; on older versions, write `Mapped[list["Thing"]]`.

### 3. Creating the tables
```python
Base.metadata.create_all(engine)
```
Creates every table defined on `Base` that doesn't exist yet.

### 4. Adding rows
```python
with Session(engine) as session:
    new_person = Person(name="Matt", age=25)
    session.add(new_person)
    session.flush()  # inserts the new person into the table, but does not commit yet (so we commit either both the new person with the new item, or none)

    new_thing = Thing(description="TV", value=1399.99, owner=new_person.id)
    session.add(new_thing)
    session.commit()
    print([t.description for t in new_person.things])  # ['TV']
    print(new_thing.person.name)                       # Matt
```
- `Session(engine)` - a workspace that tracks our objects and writes the SQL for them. `with` closes it at the end.
- `Person(...)` - only a Python object; nothing is in the database yet.
- `session.add(...)` - marks the object to be saved.
- `session.flush()` - sends the `INSERT` now, inside the open transaction. We need it because `new_person.id` is assigned by the database, and the thing needs that ID for `owner`.
- `session.commit()` - makes everything permanent. Before it, nothing is saved for good.
- After the commit, the relationships work in both directions: `new_person.things` and `new_thing.person`.

### 5. Selecting
```python
    result = session.execute(select(Person.name, Person.age)).all()
    print(result)  # [('Matt', 25)]
```
Selecting columns returns **rows**, like in Core.

```python
    people = session.scalars(select(Person).where(Person.age > 30)).all()
    print([p.name for p in people])  # [] - Matt is 25
```
Selecting a whole class returns rows that each wrap one object: `[(Person,), ...]`. `session.scalars(...)` unwraps them into the objects themselves: `[Person, ...]`. `.where(...)` filters, like `WHERE` in SQL.

### 6. Deleting
```python
    deleted = session.execute(delete(Thing).where(Thing.value < 50))
    session.commit()
    print(deleted.rowcount)  # 0 - the TV is worth more than 50
```
A `DELETE` returns no rows, only a result object. Its `rowcount` says how many rows were deleted.

### 7. Updating
```python
    updated = session.execute(
        update(Person).where(Person.name == "Matt").values(name="Matthew")
    )
    session.commit()
    print(updated.rowcount)  # 1
```
The same idea: `rowcount` says how many rows were changed.

### 8. Joins
```python
    result = session.execute(
        select(Person.name, Thing.description).join(Person.things)
    ).all()
    print(result)  # [('Matthew', 'TV')]
```
`.join(Person.things)` joins through the relationship, so SQLAlchemy already knows the condition (`things.owner = people.id`). It's an inner join: people without things are left out.

### 9. Grouping
```python
    result = session.execute(
        select(Thing.owner, func.sum(Thing.value))
        .group_by(Thing.owner)
        .having(func.sum(Thing.value) > 50)
    ).all()
    print(result)  # [(1, 1399.99)]
```
- `func.sum(...)` - calls the SQL function `SUM`; `func.<name>` works for any SQL function (`COUNT`, `AVG`, ...).
- `.group_by(...)` - one result row per owner.
- `.having(...)` - keeps only groups whose total is over 50; like `WHERE`, but applied after grouping.
