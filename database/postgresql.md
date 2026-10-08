# PostgreSQL
A powerful, open source object-relational database system.

## Install with Docker
Define a Docker Compose file:
```yaml
services:
  db:                                    # the service name; inside Compose, Postgres is reachable as "db"
    image: postgres:17                   # official Postgres image, version 17
    environment:                         # settings the Postgres image reads on first start
      POSTGRES_USER: postgres            # database user to create
      POSTGRES_PASSWORD: postgres        # its password; fine locally, never on the server
      POSTGRES_DB: postgres              # database to create
    ports:
      - "127.0.0.1:5432:5432"            # reachable from my laptop only, on Postgres's standard port
    volumes:
      - db_data:/var/lib/postgresql/data # where Postgres stores its files; kept in a named volume

volumes:
  db_data:                               # declare the named volume, so the data survives restarts
```
And run the container with `sudo docker compose -f <yaml file> up -d`.

The `environment` values are used only on the **first** start, when the volume is still empty. Changing them later does nothing, because the database already exists. To start over with new values, delete the volume too: `sudo docker compose -f <yaml file> down -v` (this deletes all data).

Always pin the version (`postgres:17`, not `postgres`). A new major version can't read the data files of an older one.

## Connection URL
Programs connect to Postgres with a URL:
```
postgresql://<user>:<password>@<host>:<port>/<database>
postgresql://postgres:postgres@localhost:5432/postgres
```
`<host>` is `localhost` from my laptop, and the service name (`db`) from another container in the same Compose project.

## psql
To use the terminal of the database, run `sudo docker compose -f <yaml file> exec db psql -U <POSTGRES_USER> -d <POSTGRES_DB>` - `exec db` runs a command inside the running db container, and `psql` is Postgres's command-line client. Here we can run the following commands:
- `\l` - list databases
- `\c <database>` - connect to another database
- `\d` - list all relations (tables, views, sequences)
- `\dt` - list all tables
- `\d <table>` - show a table's columns and types
- `\du` - list users (roles)
- `\x` - toggle expanded output (one column per line; handy for wide rows)
- `\! clear` - clear the screen
- `\q` - quit

We can also run SQL commands directly. Each one ends with `;` - without it, `psql` waits for more input.

## Basic SQL
```sql
CREATE TABLE applications (                   -- create a table
    id SERIAL PRIMARY KEY,                    -- unique ID, assigned automatically
    name TEXT NOT NULL,                       -- NOT NULL: the value is required
    created_at TIMESTAMPTZ DEFAULT now()      -- time with time zone; filled in automatically
);

INSERT INTO applications (name) VALUES ('Jānis');            -- add a row

SELECT * FROM applications;                                  -- all rows, all columns
SELECT id, name FROM applications WHERE id = 1;              -- chosen columns, filtered rows
SELECT * FROM applications ORDER BY created_at DESC LIMIT 10; -- newest 10

UPDATE applications SET name = 'Anna' WHERE id = 1;          -- change rows
DELETE FROM applications WHERE id = 1;                       -- remove rows

TRUNCATE applications;                                       -- remove all rows, keep the table
DROP TABLE applications;                                     -- delete the whole table
```
`UPDATE` and `DELETE` without `WHERE` change **every** row.

`TRUNCATE` vs `DELETE`: `TRUNCATE` empties the whole table at once and is much faster on big tables; `DELETE` removes rows one by one and can be filtered with `WHERE`. `TRUNCATE ... RESTART IDENTITY` also resets the ID counter to 1.

## Relations between tables
A **foreign key** links a row to a row in another table. Here each application belongs to one group course:
```sql
CREATE TABLE groups (
    id SERIAL PRIMARY KEY,
    category TEXT NOT NULL,                       -- e.g. 'B'
    starts_on DATE NOT NULL
);

CREATE TABLE applications (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    group_id INTEGER REFERENCES groups (id)       -- foreign key: must be an existing groups.id, or NULL
);
```
Postgres enforces it: inserting an application with a `group_id` that doesn't exist fails, and so does deleting a group that applications still point to.

## Joins
A **join** combines rows from two tables, matched by a condition (usually the foreign key):
```sql
SELECT a.name, g.category, g.starts_on           -- columns from both tables
FROM applications a                                -- "a" is a short alias for applications
JOIN groups g ON g.id = a.group_id;               -- match each application to its group
```
- **`JOIN`** (= `INNER JOIN`) - only rows that have a match in both tables. Applications without a group are left out.
- **`LEFT JOIN`** - all rows from the left table (`applications`), with `NULL` in the group columns where there is no match.

## Grouping
`GROUP BY` collapses rows with the same value into one row, and aggregate functions compute one value per group:
```sql
SELECT g.category, COUNT(*) AS applications       -- one row per category, with a count
FROM applications a
JOIN groups g ON g.id = a.group_id
GROUP BY g.category
HAVING COUNT(*) > 5                                 -- keep only groups with more than 5 applications
ORDER BY applications DESC;
```
- Aggregate functions: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`.
- `WHERE` filters rows **before** grouping; `HAVING` filters groups **after** grouping.
- Every selected column must be either in `GROUP BY` or inside an aggregate function.
