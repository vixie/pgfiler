# pgfiler

`pgfiler` copies a file into a PostgreSQL column, or a column into a file. The caller supplies the table, the key column, the key, and the value column. `pgfiler` does not derive the key from the file name.

## Build

To build `pgfiler`, we need the `libpq` headers and library, and the `pg_config` that reports their locations. Neither is part of a default system, so install one of these first:

| System | Package | Command |
|--------|---------|---------|
| macOS, Homebrew | `libpq` | `brew install libpq` |
| Debian, Ubuntu | `libpq-dev` | `apt install libpq-dev` |
| RHEL, Rocky | `libpq-devel` | `dnf install libpq-devel` |

Then run `make`.

On Linux, the packages above put `pg_config` in `/usr/bin`, where the Makefile finds it. Homebrew's `libpq` is keg-only and stays off `PATH`, so name it explicitly:

    make PG_CONFIG=$(brew --prefix libpq)/bin/pg_config

A full PostgreSQL installation also supplies `pg_config` and will also work. However, it is not otherwise needed to build, since `pgfiler` is a client and the server can be anywhere.

The Makefile sets `CC` to `clang` and takes the `libpq` include and library paths from `pg_config`. Use `make PG_CONFIG=/path/to/pg_config` whenever `pg_config` is not on `PATH`. `CBUILD` holds GCC-compatible warning flags, accepted by both `gcc` and `clang`, and `-Werror`; override `CBUILD` for a compiler that rejects them. `LDFLAGS` sets a library search path but no run-time path, so a `libpq` outside the system search path needs `LD_LIBRARY_PATH` or an equivalent at run time. There is no install target.

The client needs `libpq` 9.0 or later for `PQescapeIdentifier`, or 9.2 or later for a `postgresql://` argument to `-d`. The server needs 7.4 or later for the extended query protocol, and 9.5 or later for `upsert`.

## Usage

    pgfiler: usage error (<message>)
    usage:
    	pgfiler [-b] [-M tsf] [-T tmpdir] [-x tracelevel]
    		[-d dbname] [-h dbhost] [-p dbport] [-o pgoptions]
    		[-u pguser] [-t pgtty] [-P pgpasswd]
    		<op> <tbl> <kf> <k> <vf> [<file>]
    where:
    	<op> is select|upsert|insert|replace|append;
    	<file> defaults to stdin (put) or stdout (select);
    	-b means <vf> is bytea rather than text;
    	-M names a column to set to the file's mtime;
    	-P is visible in ps(1); PGPASSWORD or ~/.pgpass is not;
    	a <k> beginning with '-' needs a '--' before <op>;
    example:
    	pgfiler select file_table filename 1.2.3.4 file_data

### Arguments

| Argument | Meaning |
|----------|---------|
| `op`     | `select`, `upsert`, `insert`, `replace`, or `append`. |
| `tbl`    | Table name. May be schema-qualified (`schema.table`). |
| `kf`     | Key column. |
| `k`      | Key value. |
| `vf`     | Value column. |
| `file`   | Input file for the write operations, output file for `select`. Defaults to standard input or standard output. |

Where `getopt` permutes its arguments, as it does with the GNU C library, any positional argument beginning with `-` is taken as an option. Put `--` before `op` in that case. It is harmless elsewhere.

### Operations

| Operation | Statement | Notes |
|-----------|-----------|-------|
| `select`  | `SELECT vf FROM tbl WHERE kf = $1` | Requires exactly one row and a non-NULL value. A `bytea` value requires `-b`. |
| `upsert`  | `INSERT INTO tbl (kf, vf) VALUES ($1, $2) ON CONFLICT (kf) DO UPDATE SET vf = $2` | `kf` needs a unique constraint or index. |
| `insert`  | `INSERT INTO tbl (kf, vf) VALUES ($1, $2)` | Fails if the key exists and `kf` is unique. |
| `replace` | `UPDATE tbl SET vf = $2 WHERE kf = $1` | Fails if no row matches. |
| `append`  | `UPDATE tbl SET vf = COALESCE(vf, '') \|\| $2 WHERE kf = $1` | Fails if no row matches. A NULL value is treated as empty. |

With `-M tsf`, each statement also sets `tsf`. The timestamp is literal SQL text, not a parameter.

A write that affects zero rows is reported as an error.

### Options

| Option | Meaning |
|--------|---------|
| `-b` | Declare the value parameter as `bytea` rather than `text`, and request binary results for `select`. The value is sent in binary format either way. |
| `-d dbname` | Database. Default: `$USER`, then `$LOGNAME`, then the name from the password database. `PGDATABASE` is not consulted. `libpq` reads an argument containing `=`, or beginning with a connection URI prefix, as a connection string. |
| `-h dbhost` | Server host. |
| `-p dbport` | Server port. |
| `-u pguser` | Database user. |
| `-P pgpasswd` | Password. The argument is overwritten after it is read, but it is visible in `ps(1)` until then. Prefer `~/.pgpass`. The `libpq` documentation discourages `PGPASSWORD`. |
| `-o pgoptions` | Command-line options sent to the server. |
| `-t pgtty` | Passed to `libpq`, which ignores it. |
| `-M tsf` | Also set column `tsf`. For a regular file, its modification time to whole seconds, as `to_timestamp(N)`, which yields `timestamp with time zone`. For any other input, `'now'::TIMESTAMP`, which has no time zone. The two coerce differently into one column. Not valid with `select`. |
| `-T tmpdir` | Directory for spooled input. Default: `$TMPDIR` when set and non-empty, otherwise `/tmp`. An empty argument is a usage error. |
| `-x tracelevel` | A value above zero prints each SQL statement to standard error, in braces, before it runs. A non-numeric argument reads as zero and disables tracing. Keys and file values are not printed; the `-M` timestamp is, because it is part of the statement. |

`libpq` environment variables such as `PGHOST`, `PGPORT`, `PGUSER`, and `PGPASSWORD` supply any of the other connection parameters not given on the command line.

Use `-b` when the value column is `bytea` and omit it when the column is `text`. Three of the four combinations are caught. A `select` from a `bytea` column without `-b` is refused by `pgfiler`, because the value would arrive in its hex text form. A write to a `bytea` column without `-b` is refused by the server, which reports a type mismatch. A `select` from a `text` column with `-b` returns the same bytes. The fourth combination fails silently: a write to a `text` column with `-b` succeeds and stores the hex representation of the file, twice its length plus two characters.

The exit status is 0 on success and 1 on any reported failure.

### Examples

The table used below:

    CREATE TABLE file_table (
        filename text  PRIMARY KEY,
        contents bytea NOT NULL,
        mtime    timestamptz
    );

Store a file, then retrieve it:

    pgfiler -b upsert file_table filename /usr/bin/awk contents /usr/bin/awk
    pgfiler -b select file_table filename /usr/bin/awk contents ./awk

Store a stream:

    tar cf - src | pgfiler -b upsert file_table filename src.tar contents

Store a file and its modification time, then read the time back:

    pgfiler -b -M mtime upsert file_table filename report.pdf contents report.pdf
    pgfiler select file_table filename report.pdf mtime

The second command prints the stored time in the session's time zone. A newline follows it only when standard output is a terminal.

## Features

- Five operations: `select`, `upsert`, `insert`, `replace`, and `append`.
- Binary values, which may contain any byte.
- Text values, which must be valid in the client encoding and must not contain a zero byte.
- Input from a regular file, a pipe, or a file whose `st_size` is wrong, as under Linux `/proc`. Only a regular file is mapped directly, using its `stat` size. Anything else is spooled first, so its size need not be known in advance.
- Optional modification-time column.
- Keys of any type the server can convert from text: integer, `inet`, `uuid`, and others.
- Schema-qualified table names.
- `select` into a file writes a temporary file and renames it, so a reader of the file sees the old content or the new content.
- The key and the value are bound parameters and never appear in SQL text.

## Background/Theory

A file is a named sequence of bytes. Unix programs read and write files through paths and pipes. A row addressed by a key is the same abstraction behind a different interface: a name and a value. `pgfiler` maps one onto the other. A program that reads or writes files can then read or write a database without linking a database client library.

Each invocation uses one key and one whole value. A database that distributes rows places them by a partition key. A statement whose qualifier names that key can be confined to one partition, needs no coordination with the others, and costs an index lookup plus the size of the value. A statement that scans, joins, or updates many rows has none of these properties. Whole-file reads and writes by name are single-key operations, so they fit the access pattern that horizontal scaling supports.

Two qualifications apply to a partitioned table. `pgfiler` always passes the key as a parameter, so pruning happens at execution time, which PostgreSQL supports for `SELECT` from version 11 and for `UPDATE` from version 14. `upsert` additionally requires that the unique index naming `kf` include every partition key column.

Three more properties follow from the mapping.

**Atomicity.** Each invocation is one statement in its own transaction, and PostgreSQL applies a statement atomically. A concurrent reader sees the old value or the new value, never a mixture. `append` is a read-modify-write inside one statement: under the default READ COMMITTED level PostgreSQL re-evaluates the row it is updating, so concurrent appends do not lose an update. Under REPEATABLE READ or SERIALIZABLE the statement can instead fail with SQLSTATE 40001, and `pgfiler` does not retry. On the file side, `select` into a file writes a temporary file and renames it over the target.

**Idempotence.** Repeating `upsert` or `replace` with the same input leaves the same value in the value column, so a failed pipeline stage can be run again without first checking what happened. It does not leave the same row: each run writes a new row version and fires any update trigger, and with `-M` on input that is not a regular file the timestamp column is set to the current time and changes every run. `replace` fails if the row has been deleted. `insert` fails on a repeat when the key column is unique. `append` changes the value each time, and is not an incremental write: PostgreSQL rewrites and logs the whole new value, so appending to a large value costs the size of the result.

**Change detection.** With `-M`, the row holds the file's modification time. A caller can compare stored and local times to find files whose times differ. Equal times do not prove equal contents, and `pgfiler` does not perform the comparison.

The scalability belongs to the database behind the connection. `pgfiler` opens one connection, sends one statement, and exits. Partitioning, replication, and distribution are properties of the server. Beyond the statements above, `pgfiler` uses only what `libpq` does on its behalf: connection startup, authentication, and the extended query protocol.

## Architecture

`pgfiler` is one C source file of 761 lines. It depends on `libpq` and the C library. It has no threads, installs no signal handlers, and reads no configuration file of its own, although `libpq` reads `~/.pgpass` and the service file.

| Function | Role |
|----------|------|
| `main`     | Chooses the default database name, then parses options, validates the operation, connects with `PQsetdbLogin`, dispatches, disconnects, and clears the password. The database name is chosen before the options are read. |
| `usage`    | Prints the message and synopsis, then exits. |
| `get`      | Runs the `SELECT`, checks the shape of the result, and writes the value. |
| `put`      | Reads the input, builds the statement for the operation, and runs it. |
| `quote`    | Escapes an identifier with `PQescapeIdentifier`, one dotted part at a time. |
| `writeall` | Writes a whole buffer, retrying after interruption and partial writes. |
| `xasprintf`, `xstrdup` | Exit on allocation failure. |
| `xmemset`  | A `memset` the compiler cannot remove. It clears the password. |

### Writing to the database

    file or standard input
      |
      +-- regular file, size > 0 -----------------------> mmap
      |
      +-- pipe, /proc, empty regular file, mmap failure -> unlinked temporary
      |                                                   file in tmpdir -> mmap
      v
    parameter $2, binary format, explicit length -> PQexecParams -> server

The value is always sent in binary format. `-b` selects the declared type of the parameter, not the wire format. Binary is used because `libpq` derives the length of a text-format parameter with `strlen`, which is wrong for data that contains a zero byte. `libpq` also needs the whole value in one buffer before it can send, so input whose size is not known in advance is first copied to a temporary file, which is unlinked as soon as it is open. An oversize regular file is rejected from its `stat` size alone and is never spooled. Spooling stops as soon as the input exceeds the limit, so `tmpdir` receives at most that limit.

### Reading from the database

    PQexecParams(SELECT) -> check: one row, not NULL, not bytea without -b
      -> standard output
      -> or mkstemp("file.XXXXXX") -> write -> fchmod -> close -> rename(file)

When `<file>` is provided, the temporary file is in the same directory as the target, so the rename does not cross a file system. A failed query, write, `fchmod`, `close`, or `rename` removes the temporary file and leaves an existing target unchanged. An existing target's mode is copied verbatim onto the new file, including any execute, setuid, setgid, or sticky bit, and the new file is owned by the invoking user. A target that does not exist yet gets `0666` minus the umask, so a retrieved executable is not executable until its mode is set. The result of `close` is checked because some file systems report a write error only there. A newline is added only when the value is text, is not empty, does not already end in one, and the output is a terminal.

Output to standard output has none of this protection. It is written directly, is not closed by `pgfiler`, and a failed write leaves partial data.

## Limits

- `pgfiler` rejects input larger than 1073741758 bytes minus the length of the key, which follows from the server's maximum message size. PostgreSQL separately limits a `text` or `bytea` field to 1 GB, and repeated `append` reaches that limit first. Larger objects need a different tool.
- Each operation holds at least one complete copy of the value in memory. If `tmpdir` is a memory file system, a spooled input costs that much again.
- Each invocation opens one connection and runs one statement. The connection cost is paid every time.
- `put` maps the input file and does not take a snapshot. A file modified during the transfer can be stored inconsistently, and a file truncated during the transfer can terminate `pgfiler` with SIGBUS.
- When the input is mapped, the whole file is used regardless of the descriptor's current offset. When it is spooled, copying starts at the current offset.
- The spool loop does not retry an interrupted `read`, so a signal during a long transfer from a pipe can abort it.
- `select` into a file replaces it with a new file. Ownership, ACLs, and hard links are not preserved, a symbolic link is replaced by a regular file, and nothing is flushed to disk before the rename. A process killed by a signal leaves `file.XXXXXX` behind. Two concurrent retrievals into one target race, and the last rename wins.
- Do not name a device, a FIFO, or a path with a trailing slash as `<file>`. Use shell redirection instead.
- `select` to standard output dies on SIGPIPE if the reader closes early, and leaves any temporary file behind.
- A retrieved file's modification time is not restored.
- The key column should be unique. `select` fails when more than one row matches, and `replace` and `append` change every matching row.
- If neither `USER` nor `LOGNAME` is set and the user has no password-database entry, `pgfiler` exits before it reads its options, so `-d` cannot rescue it.
- This repository has no automated tests.

## Benchmark

`benchmark.sh [tree [dbname]]` creates a database, `pgfiler_bench` by default, holding the first two columns of the table shown under Examples. It then runs `pgfiler -b upsert` for every regular file under `tree`, `/usr/bin` by default, whose other-read permission bit is set, and prints one dot per stored file. If `createdb` fails, for example because the database exists, the script suggests `dropdb` and exits with status 1. It runs under `set -e`, so the first `pgfiler` failure stops it mid-line. It prepends the current directory to `PATH`, so `pgfiler` and every other command it runs, including `createdb`, `psql`, and `find`, are looked for there first. It reports no timings; use `time ./benchmark.sh` to measure.

## Author and license

`pgfiler` is owned and maintained by Paul Vixie. He worked on BIND as a software engineer at Digital Equipment Corporation from 1988 to 1993, wrote Vixie cron, and co-founded the Internet Software Consortium, renamed Internet Systems Consortium in 2004, and the Mail Abuse Prevention System.

The license is a permissive ISC-style license (there is no separate LICENSE file). The two notices in the tree are not identical.
- `pgfiler.c` carries the current ISC wording, granting use, copy, modification, "and/or" distribution, and disclaims on behalf of the Mail Abuse Prevention System.
- The Makefile carries the older wording, "and" distribution, and names the Internet Software Consortium.
