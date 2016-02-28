---
layout: post
title: Copying Big MySQL Tables Easily
permalink: /:categories/:year/:month/:day/:title.html
categories:
- mysql
---

TL;DR: There is a script to automate these steps at
[https://gist.github.com/rcmachado/ef5d57a1718f1feb0858][script] (but read this
is still recommened)

---

This is a relative common scenario: you have your production database
up and running. Your project gets bigger and you realize that you need
an updated copy of your production database on your staging environment.

The most obvious choice would use `mysqldump`. Probably something like
this:

```bash
mysqldump --single-transaction db_prod | gzip -9 > db_prod.sql.gz
# (copy the file to staging machine)
gunzip -c db_prod.sql.gz | mysql db_staging
```

Or if you are restoring to the same server you can skip the gzip part
(let's assume everybody knows that its a bad idea™ to use staging and
production on the same machine and leave this here only for teaching
purposes):

```bash
mysqldump --single-transaction db_prod | mysql db_staging
```

After some minutes (hours?) staging database will be an consistent copy
of production.

## What if your backup take hours?

As your database gets bigger, your backup also takes more time.
Depending on server configuration, a not so big database with ~20Gb
could take hours to copy with a traditional `mysqldump`. This is
speacially problematic on cloud environments - when "disk" performance
suffer with latency problems.

Depending how your system was configured, you can use [LVM][] to take
snapshots of database volume. Although this is the best option,
sometimes isn't affordable one (because of lack of resources or
legacy systems).

## xtrabackup to the rescue

At the old days of MySQL, the only reliable way to make a backup of
InnoDB tables was to use a program called InnoDB Hot Backup - a
proprietary (and paid) one.

Recently, I discovered that the guys from [Percona][] built on open
source tool with the same purpose of old InnoDB Hot Backup. It's called
[xtrabackup][]. It's a very good tool that makes easy to make full,
partial or incremental backups (but that is a subject for another post).

One thing that is possible with xtrabackup is to restore individual
tables: as the backup is basically the data files copied from MySQL's
own datadir (and the transaction log replayed to have a consistent
snapshot of database), this is **much** faster than extract data as
text and import it again (this was, basically, what we did with the
`mysqldump | mysql` commands before).

But there is some requirements: this only works for InnoDB tables, you
need to have enabled the option `innodb_file_per_table` and your MySQL
version should be at least 5.6.

## Hands on!

To start, let's make a full backup of our data. We're interested only
in tables in the database _db_prod_:

```bash
xtrabackup --backup --tables="^db_prod[.].*" --target-dir=/tmp/our-backup
```

After that, we need to "prepare" this backup to be restored as .ibd
files:

```bash
xtrabackup --prepare --export --target-dir=/tmp/our-backup
```

After this step, we'll have 4 files for each table: the .frm (that
contains information about the table structure), *.exp (for their of
MySQL, [XtraDB][]), *.cfg (for MySQL 5.6) and .ibd (our data file).

Make sure the tables on the destination database (in our case,
db_staging) have **exactly the same structure** that the ones you're
importing from db_prod. The best way to archive this is to simply copy
over the structure from db_prod:

```bash
mysqldump -d db_prod | mysql db_staging
```

Before copying the exported data files to db_staging, we need to
discard the tables tablespaces. For each table you want to restore, do:

```sql
ALTER TABLE db_staging.mytable DISCARD TABLESPACE;
```

Copy the corresponding .idb and .exp/.cfg files (in our case,
mytable.ibd and mytable.exp/mytable.cfg) to the database directory
inside the MySQL datadir (let's assume our datadir is /var/lib/mysql):

```bash
sudo rsync -a /tmp/our-backup/mytable.{ibd,exp,cfg} /var/lib/mysql
```

Note: we used `rsync -a` to preserve permissions. If you use `cp` or
`mv`, remember to fix the file permissions or you'll get an error when
importing the tablespaces.

After copying the files (and fixed the permissions, if necessary), we
can import the tablespaces again (this will make MySQL use our recently
copied idb files):

```sql
ALTER TABLE db_staging.mytable IMPORT TABLESPACE;
```

## Voilà

You now have copied a table from one database to the other - and in a
fraction of time than if you have used `mysqldump`. This is possible
because our overhead was minimal (basically, we just copied some files
around) - on the other way, exporting to SQL and importing again has a
huge overhead to convert data from binary form to SQL, parse the SQL
and convert it again to binary again.

## But as we're all lazy

I've made a small shell script to automate the [export/import tables][script]
process. As with any script you'll use, read it carefully and see if
it's doing what you need. Use it at your own risk (but I used it
myself ;)).


[LVM]: http://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)
[Percona]: http://www.percona.com/
[xtrabackup]: http://www.percona.com/software/percona-xtrabackup
[XtraDB]: http://www.percona.com/software/percona-xtradb
[script]: https://gist.github.com/rcmachado/ef5d57a1718f1feb0858
