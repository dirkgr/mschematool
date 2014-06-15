mschematool
===========

mschematool is a simple tool for managing database migrations. Migrations are either SQL files or Python modules. They are ordered linearly using just lexicographical ordering of file names (with timestamp being a leading component). Information about executed migrations is stored inside a database for which migrations were executed in a simple table with file names only. The lightweight design enables changing the table contents manually, if necessary, and generally the tool tries to not enforce any hard rules.

Currently only PostgreSQL is supported.

Installation
============
The tool is available as a Python package, so the simplest method to install it is using `pip` (or `easy_install`):
```
$ pip install mschematool
```

Configuration
=============
Configuration file is a Python module listing available database and migration files locations:

```
DATABASES = {
        'default': {
            'migrations_dir': './migrations/',
            'engine': 'postgres',
            'dsn': 'host=127.0.0.1 dbname=mtutorial',
        },

        'other': {
            'migrations_dir': './migrations_other/',
            'engine': 'postgres',
            'dsn': 'host=127.0.0.1 dbname=mother',
            'after_sync': 'pg_dump -s mother > /tmp/mother_schema.sql',
        },
}

LOG_FILE = '/tmp/mtest1.log'
```

We have two "dbnicks" (short database names): `default` and `other`. For each we specify options:
* `migrations_dir` is a directory with migrations files (note that it's usually not a good idea to use a relative path here).
* `engine` specifies database type
* `dsn` specifies database connection parameters for the `postgres` engine, as described here: http://www.postgresql.org/docs/current/static/libpq-connect.html#LIBPQ-CONNSTRING
* `after_sync` optionally specifies a shell command to run after a migration is synced (executed). In the case of `other` database a schema dump is performed.
* `LOG_FILE` optionally specifies a log file which will record all the executed SQL and other information useful for debugging.


Path to a configuration module can be specified using `--config` option or `MSCHEMATOOL_CONFIG` environment variable:
```
$ export MSCHEMATOOL_CONFIG=./config_tutorial.py
```
(again, it's better to use an absoulte path so the `mschematool` command will work from any directory).

Doing work
==========
Assuming the `mtutorial` database is created, we first need to initialize the database - create table `migration` for storing names of executed migrations.
```
$ mschematool default init_db
```
All commands are specified this way - the first argument is a "dbnick" from a config, the second is an actual command (run `mschematool --help` to see a short summary of commands).

Now given that we have a few migration files:
```
$ ls migrations 
m20140615132455_create_article.sql
m20140615133521_add_column_author.sql
m20140615135414_insert_data.py

```
let's sync them, checking previously that all will be synced using `to_sync`:
```
$ mschematool default to_sync
m20140615132455_create_article.sql
m20140615133521_add_column_author.sql
m20140615135414_insert_data.py

$ mschematool default sync   
Executing m20140615132455_create_article.sql
Executing m20140615133521_add_column_author.sql
Executing m20140615135414_insert_data.py

$ mschematool default to_sync
$

```
`sync` command executes all migrations that weren't yet executed. For selecting a subset of migrations, the best way is to put migrations (or symlinks to them) in different directories and specify them in a configuration module under different "dbnicks" configs.

To execute a single migration without executing all the other available for syncing, use `force_sync_single`:
```
$ mschematool default force_sync_single m20140615132455_create_article.sql
```

For more fine-grained control, modify `migration` table manually. The content is simple:
```
$ psql mtutorial -c 'SELECT * FROM migration'
                 file                  |          executed          
---------------------------------------+----------------------------
 m20140615133521_add_column_author.sql | 2014-06-15 19:19:42.100535
 m20140615135414_insert_data.py        | 2014-06-15 19:19:42.101006
(2 rows)
```

For example, to forget about execution of a given migration, run
```
$ psql mtutorial -c "DELETE FROM migration WHERE file like '%insert_data%'"
DELETE 1
```
