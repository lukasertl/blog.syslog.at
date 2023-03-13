---
title: "Transaction ID Wraparound"
date: 2023-03-13T18:41:29+01:00
---
I was on call this weekend, and on Sunday morning I received some
reports that one of our PostgreSQL clusters was acting up.

Since our regular server monitoring hadn't notified me of anything I was
expecting that this would be some temporary connection problems or
something like that.

Oh boy, was I wrong.

I logged into the offending machine and checked the PostgreSQL logs, and
I was *flooded* with error messages:

```
WARNING:  database "my_prod_db" must be vacuumed within 1000000 transactions
HINT:  To avoid a database shutdown, execute a database-wide VACUUM 
  in that database.
ERROR:  database is not accepting commands to avoid wraparound data loss
  in database "my_prod_db"
HINT:  Stop the postmaster and vacuum that database in single-user mode.
```
So basically the database cluster was read-only, and I had no idea why
and more importantly, how to fix that.

I started googling, and soon found out that we had maneuvered
ourselves into a dreaded situation called "Transaction ID Wraparound", 
something you should basically avoid at all costs.

Awesome, exactly what I needed on a Sunday.

With the help of pages like
[this](https://www.percona.com/blog/overcoming-vacuum-wraparound/) or
[this](https://www.crunchydata.com/blog/managing-transaction-id-wraparound-in-postgresql)
I started the recovery process, shutting down the PostgreSQL service,
starting the affected database in single-user mode and vacuuming it.  Of
course, as this was one of our larger databases with almost 1 TB, it
would take hours and hours to finish.  Since I had no progress meter I
resorted to checking the modification dates of the database files
themselves, to give me an estimated how long we would be offline (hint:
*very* long).

Naturally, the vacuuming would generate lots of WALs, so of course I ran
into disk space problems and had to start all over again.

While I was waiting for the vacuum to finish I tried to find out how in
the world this was able to happen at all.  Basically if your autovacuum
jobs run correctly you shouldn't end up in this situation.  Our
databases do have autovacuuming configured, but apparently there was a
corrupt index that prevented the autovacuum job to do it's work, leading
to a pileup of unvacuumed tables, leading to the wraparound of
transaction IDs.

I remember that I reported this exact index to the responsible
developers some time ago, but didn't appreciate the gravity of the
situation - I certainly would have stressed that they fixed it!

Well, lessons learned.
