# mysqlexportcsv
Export mysql data the fastest way on server side

# Usage
```
Usage: mysqlexportcsv [-Fsd] [-Dupt <STR>] [db1[.table1] [db2[.table2] ...]]
 Exports all tables unless db or table names are given
Flags:
 -F, --flush   Flush all tables before lock
 -s, --struct  Export only structure
 -d, --data    Export only data
Options:
 -D, --DSN     DSN (eg. "dbname=mysql;host=localhost;port=3306")
 -u, --user    Mysql username (env: DBI_USER)
 -p, --pass    Mysql password (env: DBI_PASS)
 -t, --target  Directory to export data on server side
```

# Theory
This script utilizes MySQL's ``SELECT INTO OUTFILE`` capability
to save table data into CSV files, each table of each database  into
separated files. One can issue the ``LOAD DATA INFILE`` in loop to
restore data on the same mysql instance or on other.

# CSV can hold only text data. Isn't it?
No. MySQL properly escapes and de-escapes binary data, newlines and 
meta chars stored in CSV.

# Features
* mysqlexportcsv saves table structures too, however
in SQL-format and on client-side (where script runs) into
the working directory.
* If no parameters are given, then all tables in all databases 
are going to be exported (except performance_schema and mysql.%_log).
* If a parameter does not include a dot, then it treated as a database
and all of its tables are going to be exported.

# Restore data example
```
USE dbname;
DROP TABLE IF EXISTS tname;
source dbname.tname.struct.sql;
LOAD DATA INFILE 'dbname.tname.data.csv'
  INTO TABLE tname
  FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '\"' ESCAPED BY '\\'
  LINES TERMINATED BY '\n';
```

# Caveats

## File modes
By default, mysqlexportcsv asks mysql to export data directly in 
mysql's data directory. You can override it, but mysql daemon's
user must can write the target directory and also, exported 
files will be **world-writable** - this is how mysql works internally.
It does not mean information leak in common environments, due to
mysql's data directory is not accessible by others. You may
create the custom target directory by ``install -d /path/to/mysql/export 
-o mysql -m 0700``.

## Locking and atomic operation
It uses ``LOCK TABLES READ`` to ensure data consistency. But it is still
possible to miss one or more tables from exporting when a table is
created between enumerating tables and locking them.
Use ``--flush`` flag when tables created or dropped frequently. With it,
mysqlexportcsv issues ``FLUSH TABLES WITH READ LOCK`` which locks
globally, so table and database names can be enumerated comfortably.
The only drawback of ``--flush`` is the unneccessary I/O.
