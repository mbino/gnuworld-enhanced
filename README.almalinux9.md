# Building on AlmaLinux 9 with PostgreSQL 12 or newer

This branch (`almalinux9`) is gnuworld-enhanced as run by ChatBox.nu, made to build and run on
AlmaLinux 9 (GCC 11, OpenSSL 3) with PostgreSQL 16.

## Changes compared to `master`

- PostgreSQL 12 removed the `abstime` type. Every `now()::abstime::int4` in the C++ sources and in
  `doc/*.sql` is replaced by `floor(extract(epoch from now()))::int4`, which gives the same value
  (the current Unix time, rounded down to whole seconds).

## Build

```
dnf install gcc gcc-c++ make autoconf automake libtool libtool-ltdl-devel \
    openssl-devel postgresql-server libpq-devel git
CXX="g++ -std=gnu++11" ./configure --with-pgsql-home=/usr --with-extra-includes=/usr/include/pgsql \
    --enable-modules=cservice,ccontrol,dronescan,openchanfix
make
make install
```

`CXX="g++ -std=gnu++11"` is needed because GCC 11 compiles as C++17 by default, which does not allow
the dynamic exception specifications (`throw(...)`) used in this code. Passing it through `CXXFLAGS`
does not work: `configure` overrides `CXXFLAGS`.

## Moving an existing database from PostgreSQL 9.x

Dump the old cluster with `pg_dumpall`, then replace the removed type in the dump before restoring:

```
sed -e "s/((now())::abstime)::integer/(floor(date_part('epoch'::text, now())))::integer/g" \
    -e "s/now()::abstime::int4/floor(extract(epoch from now()))::int4/g" \
    dumpall.sql > dumpall-pg16.sql
psql -f dumpall-pg16.sql postgres
```

The first pattern is used in column defaults, the second in the trigger functions (`new_user`,
`delete_user`, `delete_channel`, `delete_level`, `delete_ban`). Without this the triggers fail only
when they run, for example when a new user registers.

Keep the existing `SQL_ASCII` encoding of the databases (create the cluster with `--locale=C`): the
data mixes Latin-1 and UTF-8 and converting it can damage characters.
