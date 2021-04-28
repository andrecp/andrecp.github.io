---
layout: post
title:  "A simple migration tool in python"
date:   2020-09-06 08:55:38 -0700
tags: python software
---

## A simple migration tool in python

In doing a side project I started to evaluate how to best manage the life cycle of the database schema, in this particular project I am trying to avoid dependencies and to use KISS (keep it stupid simple) as much as possible. 

It's a REST API written in python3.8 and the only dependencies so far are for [passlib](https://passlib.readthedocs.io/en/stable/), [jwt](https://pyjwt.readthedocs.io/en/latest/) and [tornado](http://tornadoweb.org/en/stable/web.html). It uses [sqlite](https://www.sqlite.org/index.html) as the database.

In the past I have used [alembic](https://pypi.org/project/alembic/) with [SQLAlchemy](https://www.sqlalchemy.org/) as an ORM and it works really well! [Django migration](https://docs.djangoproject.com/en/3.1/topics/migrations/) is also something I have used and is solid. But... no ORM this time.

When evaluating what's out there [flywaydb](https://flywaydb.org/) looked interesting but I would have to manage java dependencies... I almost ended up using [migrate](https://github.com/golang-migrate/migrate) but decided to write my own.

### Database migrations 101

Ultimately for database migrations we want to go from **state A** of the database to **state B**. We do so by executing SQL statements against the database.

When changing state we can do things like
* Create a new table
* Modify a type of a column
* Modify data

In order to manage the life cycle of a database we need to store the current version of the database somewhere. 
We then need to, when applying a SQL migration script, know it is the right one to apply against the current version of the database.

### Folder and files structure

The root folder in my project looks roughly like this:

```
bin/      # Binary part of a release, like, the command to run the webapp.
python/   # The code that implements the REST API.
scripts/  # Has scripts to run against my project, not part of a release.
sql/      # SQL migrations.
```

### create_db

For simplicity's sake I created a separate script just to create the database, it will also be the first migration!

The create database migration is a file inside the `sql` folder called `0000-createdb.sql`. 

For our file name convention the first four digits are the version of the migration, after the `-` we have the human readable name and then the `.sql` extension.

We wil have a `schema_history` table which will record every migration  version applied against our database, we can see the contents of `0000-createdb.sql` here:

```sql
CREATE TABLE IF NOT EXISTS schema_history (
    applied_version INTEGER NOT NULL
);

INSERT INTO schema_history (applied_version) VALUES (0);
```

The `create_db` script will apply our first migration to the database. Put it under `scripts/`.

```python
#!/usr/bin/env python3
import os
import sqlite3

# Path to your sqlite database can be set via the envvar DATABASE_PATH
db_session = sqlite3.connect(os.getenv("DATABASE_PATH", "/tmp/mydb"))

# Get the path of our script
CREATE_DB_PATH = "/".join(os.path.realpath(__file__).split("/")[:-2])

def main():
    # Read our sql migration to create the database.
    with open(os.path.join(CREATE_DB_PATH, "sql/0000-createdb.sql")) as sql_script:
        sql_statement = sql_script.read()

    # Get a cursor and execute the SQL script.
    cursor = db_session.cursor()
    cursor.executescript(sql_statement)

    # Commit our changes
    db_session.commit()

if __name__ == "__main__":
    main()
```

#### Creating the database

```bash
# Make it executable
chmod +x ./scripts/create_db

# Execute it
./scripts/create_db

# Verify it
sqlite3 /tmp/mydb

sqlite> select applied_version from schema_history;
0
```

There we go! As we can see we have created a database and the migration version 0 was applied.

### migrate

Migrate is our second and more generic script! You can give it any migration file and it will go from the current state of the database to the one given in the migration (if allowed).

We have two rules that we validate in our `migrate` script:
1. The file given as a migration must comply to our naming schema.
2. It must be a single version higher than the current version of the database.

If our migration passes the rules we then go ahead and:
1. Apply the migration.
2. Add a record in `schema_history` with our new `applied_version`.

Here goes the `scripts/migration` script:

```python
#!/usr/bin/env python3
import argparse
import re
import os
import sqlite3

# Path to your sqlite database can be set via the envvar DATABASE_PATH
db_session = sqlite3.connect(os.getenv("DATABASE_PATH", "/tmp/mydb"))

# Regex for the filename.
MIGRATION_SCRIPT_SCHEMA = re.compile(
    r"(?P<version>^[0-9]{4})-(?P<name>\w+)(?P<extension>\.sql)"
)


def get_current_schema_version() -> int:
    """Get the latest database schema version applied."""
    cursor = db_session.cursor()
    query = """
    SELECT applied_version FROM schema_history
        ORDER BY applied_version DESC LIMIT 1;
    """
    cursor.execute(query)
    return int(cursor.fetchone()[0])


def get_version_from_filename(sql_file_name: str) -> int:
    """Extract the intended schema version from the filename."""
    match = re.search(MIGRATION_SCRIPT_SCHEMA, sql_file_name)
    if not match:
        raise RuntimeError(
            "Invalid script, must be in the format of 'DDDD-name.sql', for example sql/0001-createdb.sql"
        )
    return int(match.group("version"))


def main():
    arg_parser = argparse.ArgumentParser(
        "migrate",
        "migrate --to sql/$VERSION",
        "Migrate your database schema to a version defined in sql/",
    )
    arg_parser.add_argument(
        "-s",
        "--script",
        help="SQL script to apply, format is 'DDDD-name.sql', for example sql/0001-change_users_table.sql",
    )
    parsed_args = arg_parser.parse_args()

    # Account for relative / full paths.
    sql_script_path = parsed_args.script
    if "/" in sql_script_path:
        sql_script = sql_script_path.split("/")[-1]
    else:
        sql_script = sql_script_path

    # Extract version of current migration file and database.
    new_schema_version = get_version_from_filename(sql_script)
    current_schema_version = get_current_schema_version()

    # Do validations.
    if new_schema_version == current_schema_version:
        raise RuntimeError(
            f"You're trying to go from {new_schema_version} to {current_schema_version}."
            " Migration already applied."
        )

    if new_schema_version - current_schema_version != 1:
        raise RuntimeError(
            f"You're trying to go from {new_schema_version} to {current_schema_version}."
            " Can only upgrade one version at a time."
        )

    # Execute the migration.
    with open(sql_script_path) as sql_script:
        sql_statement = sql_script.read()
    cursor = db_session.cursor()
    cursor.executescript(sql_statement)
    cursor.execute("INSERT INTO schema_history VALUES (?)", (new_schema_version,))
    db_session.commit()


if __name__ == "__main__":
    main()
```

#### Executing a migration

Let's create a `users` table in our project, for that we need a new migration file at `sql/0001-create_users_table.sql`:

```sql
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
    username VARCHAR(255) NOT NULL UNIQUE,
    email VARCHAR(255) UNIQUE,
    password VARCHAR(255) NOT NULL,
    created DATETIME NOT NULL DEFAULT (datetime(CURRENT_TIMESTAMP, 'utc'))
);
```

Now you can execute it

```bash
# Make it executable
chmod +x ./scripts/migrate

# Execute it
./scripts/migrate --script sql/0001-create_users_table.sql

# Verify it
sqlite3 /tmp/mydb

sqlite> .schema users
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
    username VARCHAR(255) NOT NULL UNIQUE,
    email VARCHAR(255) UNIQUE,
    password VARCHAR(255) NOT NULL,
    created DATETIME NOT NULL DEFAULT (datetime(CURRENT_TIMESTAMP, 'utc'))
);
```

Voila! Our users table is created.

### Conclusion

This should be enough for this project! In my case I have some of those functions behind the `python` folder so that I can re-use, for example, to get a `dbsession`.

It is of course very far from fully featured but it should give an idea of  what other more complex tools are doing. It should also hopefully be enough for very simple use cases. 

I will update this blogpost if I ever have the need to make the tooling better for this project.
