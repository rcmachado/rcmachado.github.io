---
layout: post
title: Upgrading PostgreSQL using Homebrew
categories:
- postgres
- postgis
---

Brew really makes installation of libraries and utilities very fast -
but at a cost. Try to upgrade a more complicated software and you'll
see what I'm saying.

---

[Homebrew][] is really useful for get up & running quickly. But when
it's time to upgrade some piece of software without loosing data - like
a database for example - you have to be careful or you'll loose your
data.

With major PostgreSQL upgrades (9.3 to 9.4, for example), you have to
upgrade your database files also - or you'l simply get an error saying
`FATAL:  database files are incompatible with server`. It's really easy
to avoid that - you just need to do some little steps before doing the
usual `brew upgrade && brew cleanup`.

## Before we start

If you already upgraded postgres and got an error saying:

    LOG:  skipping missing configuration file "/usr/local/var/postgres/postgresql.auto.conf"
    FATAL:  database files are incompatible with server
    DETAIL:  The data directory was initialized by PostgreSQL version 9.3, which is not compatible with this version 9.4.0.

You will need to remove the new installation (your new version could be
different) and link to the old one:

    brew unlink postgres
    rm -rf /usr/local/Cellar/postgres/9.4.0

If you uses Postgis, take a look at the end of the post.

## Upgrading PostgreSQL

In this post, we are upgrading from 9.3 to 9.4 version - but these steps
apply to other versions also.

First of all make sure your postgres server is stopped. If you use
`launchd` to run your postgres server, unload the plist first:

    launchctl unload ~/Library/LaunchAgents/homebrew.mxcl.postgresql.plist

After that, rename your data directory to something else. If you
didn't install PostgreSQL using the default configurations, adapt the
paths bellow:

    mv /usr/local/var/postgres /usr/local/var/postgres.old

Now you can upgrade using the normal way (but don't remove old version
yet):

    brew update
    brew upgrade postgres

Now you have both versions installed - but the data files are empty.
Let's use `pg_upgrade` to copy the old files with the correct data
format (adjust your postgres versions):

    cd /usr/local
    ./Cellar/postgres/9.4.0/bin/pg_upgrade -d var/postgres.old -D var/postgres -b Cellar/postgresql/9.3.5_1/bin -B Cellar/postgresql/9.4.0/bin

Don't forget to adjust the paths to correct versions!

`pg_upgrade` will ran some checks and start migrating you data from old
directory to the new one - this shouldn't tak much time, but it depends
on the size of databases. After it finishes, startup your database
server again. If you use `launchd`:

    launchctl load ~/Library/LaunchAgents/homebrew.mxcl.postgresql.plist

Now we can run some more commands to generate optimizer statistics and
cleanup old files:

    ./analyze_new_cluster.sh
    ./delete_old_cluster.sh
    brew cleanup postgres

After that, our database will be upgraded without loosing data!

## Notes on Postgis

If you use Postgis, you will have to reinstall it after installing the
new PostgreSQL version (so it'll link to the correct version). The
easiest way to do that is to reinstall it:

    brew uninstall postgis
    brew install postgis
