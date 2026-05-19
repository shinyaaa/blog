---
title: "Analyzing psqlrc settings on GitHub: How PostgreSQL engineers configure PostgreSQL"
published_at: 2025-11-30
tags: postgres, psqlrc
---

## Introduction

Recently, I read an article titled "[Alias Settings of Engineers Around the World](https://qiita.com/reireias/items/d906ab086c3bc4c22147)" (in Japanese). As a PostgreSQL engineer, this got me thinking: "If they are customizing their bash aliases, how are they configuring their `psql` environments?"

Driven by curiosity, I decided to investigate GitHub repositories to see how developers commonly configure their `psqlrc` files.

## What is psqlrc?

`psql` is the terminal-based front-end for PostgreSQL, allowing you to execute SQL interactively. Its configuration file is named `psqlrc`, and it has the following characteristics:

* **System-wide settings:** The system-wide `psqlrc` file is stored in `../etc/` relative to the directory containing the PostgreSQL executable. This directory can be explicitly set using the environment variable `PGSYSCONFDIR`.
* **User-specific settings:** The user-specific `.psqlrc` file is stored in the executing user's home directory. This file can be explicitly set using the environment variable `PSQLRC`.
* **Versioning:** You can target specific psql versions by appending the version number to the filename, such as `.psqlrc-14`.
* **References:** For more details, refer to the [official documentation](https://www.postgresql.org/docs/current/app-psql.html) or the [PostgreSQL Wiki](https://wiki.postgresql.org/wiki/Psqlrc).

## Investigation

### Methodology

1.  Use the GitHub API to extract URLs of files containing the word `psqlrc`.
2.  Insert the contents of these files into a PostgreSQL database.
3.  Analyze the most frequently used settings.

### Extracting URLs via GitHub API

I used the [Search API](https://docs.github.com/en/rest/search). Here are the key points to consider:

* The GitHub API returns a maximum of 1,000 results. To bypass this, I sliced the search query by file size.
* The API returns only 100 results per page, so I used a `for` loop to fetch all pages.
* To avoid hitting rate limits and angering GitHub, I inserted `sleep` commands.

```bash:extract_psqlrc.sh
#!/bin/bash

GITHUB_TOKEN=hoge # Set your own GitHub token here

# API call for files 0 to 1000 bytes
for size in `seq 0 50 950`;
do
  for page in `seq 10`;
  do
    curl -ksS \
      -H "Accept: application/vnd.github.v3+json" \
      -H "Authorization: token ${GITHUB_TOKEN}" \
      -o results/${size}-$((size+49))-${page}.json \
      "[https://api.github.com/search/code?q=filename:psqlrc+size:$](https://api.github.com/search/code?q=filename:psqlrc+size:$){size}..$((size+49))&per_page=100&page=${page}"
    sleep 60
  done
done

# API call for files over 1000 bytes
for page in `seq 10`;
do
  curl -ksS \
    -H "Accept: application/vnd.github.v3+json" \
    -H "Authorization: token ${GITHUB_TOKEN}" \
    -o results/1000-1049-${page}.json \
    "[https://api.github.com/search/code?q=filename:psqlrc+size](https://api.github.com/search/code?q=filename:psqlrc+size):>1000&per_page=100&page=${page}"
  sleep 60
done

# Extract html_url
for size in `seq 0 50 1000`;
do
  for page in `seq 10`;
  do
    cat results/${size}-$((size+49))-${page}.json | \
    jq -r ".items[] | .html_url" | \
    sed "s#/blob/#/raw/#g" >> results/url_list.txt
  done
done
```

### Analyzing the Data in PostgreSQL

I analyzed the **2,005 files** obtained from the process above. Since I'm a Postgres engineer, I naturally decided to insert the data into PostgreSQL for analysis.

I'll skip the insertion script, but I stored the `psqlrc` contents line by line into a table like this:

```sql
CREATE TABLE psqlrc (
  id INT,
  line_num INT,
  statement TEXT,
  PRIMARY KEY (id, line_num)
);

SELECT * FROM psqlrc WHERE id = 1;
 id | line_num |           statement
----+----------+-----------------------------
  1 |        1 | \set QUIET 1
  1 |        2 | \x auto
  1 |        3 | \set VERBOSITY verbose
  1 |        4 | \set HISTCONTROL ignoredups
  1 |        5 | \unset QUIET
(5 rows)
```

I executed the following SQL to rank the commands based on the first word of the `statement`. (I excluded comment lines and converted text to lowercase to treat `SET` and `set` as the same).

```sql
SELECT
  lower(split_part(statement, ' ', 1)) AS command,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY command
HAVING left(lower(split_part(statement, ' ', 1)), 2) != '--'
ORDER BY count DESC
LIMIT 10;
  command  | count 
-----------+-------
 \set      |  8929
 \pset     |  2565
 \echo     |  1076
 \x        |  1018
 \timing   |   995
 \unset    |   565
 \setenv   |   273
 \r        |   207
 set       |   132
 \encoding |   123
(10 rows)
```

Here is a summary of the commands and their formats:

| Command | Format | Description |
| :--- | :--- | :--- |
| `\set` | `\set [ name [ value [ ... ] ] ]` | Sets the psql variable `name` to `value`. |
| `\pset` | `\pset [ option [ value ] ]` | Sets options affecting the output of query result tables. |
| `\echo` | `\echo text [ ... ]` | Prints arguments to standard output, separated by spaces and followed by a newline. |
| `\x` | <code>\x [ on &#124; off &#124; auto ]</code> | Toggles or sets extended table formatting mode. |
| `\timing` | <code>\timing [ on &#124; off ]</code> | Toggles or sets the display of how long each SQL statement takes to execute. |
| `\unset` | `\unset name` | Unsets (deletes) the psql variable `name`. |
| `\setenv` | `\setenv name [ value ]` | Sets the environment variable `name` to `value`. |
| `\r` | `\r` (`\reset`) | Resets (clears) the query buffer. |
| `SET` (SQL) | N/A | Changes a run-time parameter. |
| `\encoding` | `\encoding [ encoding ]` | Sets the client character set encoding. |

Let's look at each of them individually.

---

### `\set`

```sql
SELECT
  lower(split_part(statement, ' ', 1)) AS item1,
  lower(split_part(statement, ' ', 2)) AS item2,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY item1, item2            
HAVING lower(split_part(statement, ' ', 1)) = '\set'
ORDER BY count DESC
LIMIT 30;
 item1 |       item2       | count 
-------+-------------------+-------
 \set  | prompt1           |  1013
 \set  | quiet             |   919
 \set  | comp_keyword_case |   860
 \set  | histcontrol       |   823
 \set  | histfile          |   819
 \set  | prompt2           |   766
 \set  | verbosity         |   717
 \set  | histsize          |   261
 \set  | on_error_rollback |   254
 \set  | on_error_stop     |    85
 \set  | locks             |    84
 \set  | uptime            |    83
 \set  | clear             |    83
 \set  | settings          |    70
 \set  | dbsize            |    70
 \set  | activity          |    67
 \set  | tablesize         |    64
 \set  | conninfo          |    64
 \set  | waits             |    63
 \set  | sp                |    59
 \set  | uselesscol        |    59
 \set  | ll                |    57
 \set  | show_slow_queries |    54
 \set  | autocommit        |    46
 \set  | pager             |    43
 \set  | version           |    43
 \set  | echo_hidden       |    42
 \set  | menu              |    40
 \set  | tsize             |    35
 \set  | extensions        |    35
(30 rows)
```

#### `\set PROMPT1`, `\set PROMPT2`

```sql
SELECT              
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING
  lower(split_part(statement, ' ', 1)) = '\set' AND
  lower(split_part(statement, ' ', 2)) = 'prompt1'
ORDER BY count DESC
LIMIT 10;
                                statement                                | count 
-------------------------------------------------------------------------+-------
 \set PROMPT1 '%[%033[1m%]%M %n@%/%R%[%033[0m%]%# '                      |   258
 \set PROMPT1 '%[%033[1m%][%/] # '                                       |    53
 \set PROMPT1 '%[%033[33;1m%]%x%[%033[0m%]%[%033[1m%]%/%[%033[0m%]%R%# ' |    51
 \set PROMPT1 '(%n@%M:%>) [%/] > '                                       |    30
 \set PROMPT1 '%[%033[1m%]%M/%/%R%[%033[0m%]%# '                         |    24
 \set PROMPT1 '%~%x%# '                                                  |    23
 \set PROMPT1 '%M:%[%033[1;31m%]%>%[%033[0m%] %n@%/%R%#%x '              |    23
 \set PROMPT1 '%n@%M %~>'                                                |    18
 \set PROMPT1 '%M:%> %n@%/%R%#%x '                                       |    16
 \set PROMPT1 '%n@%M:%>%x %/# '                                          |    12
(10 rows)
```
```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING
  lower(split_part(statement, ' ', 1)) = '\set' AND
  lower(split_part(statement, ' ', 2)) = 'prompt2'
ORDER BY count DESC
LIMIT 10;
                   statement                   | count 
-----------------------------------------------+-------
 \set PROMPT2 '[more] %R > '                   |   305
 \set PROMPT2 ''                               |    97
 \set PROMPT2 '... > '                         |    47
 \set PROMPT2 '... # '                         |    31
 \set PROMPT2 '%M %n@%/%R %# '                 |    24
 \set PROMPT2 '%R%x%# '                        |    15
 \set PROMPT2 '%R%# '                          |    12
 \set PROMPT2 '%[%033[1;33m%]%R%#%[%033[0m%] ' |    12
 \set PROMPT2 '> '                             |    12
 \set PROMPT2 '%R> '                           |     8
(10 rows)
```

`PROMPT1`, `PROMPT2`, and `PROMPT3` are variables that configure the psql command prompt.

* **PROMPT1:** The normal prompt issued when psql requests a new command.
* **PROMPT2:** The prompt issued when more input is expected (e.g., when a command is not terminated by a semicolon or a quote is left open).
* **PROMPT3:** The prompt issued during a SQL `COPY FROM STDIN` command when row input is expected.

It seems many people customize `PROMPT1` and `PROMPT2`.

Here are examples of what these prompts look like when connected as the `rocky` user (superuser) to the `postgres` database via a Unix domain socket:

**PROMPT1 Examples**

| PROMPT1 Setting | Display Result |
| :--- | :--- |
| `%/%R%x%#` (Default) | `postgres=#` (Default) |
| `%[%033[1m%]%M %n@%/%R%[%033[0m%]%#` | `[local] rocky@postgres=#` |
| `%[%033[1m%][%/] #` | `[postgres] #` |
| `%[%033[33;1m%]%x%[%033[0m%]%[%033[1m%]%/%[%033[0m%]%R%#` | `postgres=#` (Colored) |
| `(%n@%M:%>) [%/] >` | `(rocky@[local]:5432) [postgres] >` |
| `%[%033[1m%]%M/%/%R%[%033[0m%]%#` | `[local]/postgres=#` |

**PROMPT2 Examples**

| PROMPT2 Setting | Display Result |
| :--- | :--- |
| `%/%R%x%#` (Default) | `postgres(#` (Default) |
| `[more] %R >` | `[more] ( >` |
| `''` | (No display) |
| `... >` | `... >` |
| `... #` | `... #` |
| `%M %n@%/%R %#` | `[local] rocky@postgres( #` |

It appears many users prefer to display the connection user, database, port, and hostname.

#### `\set QUIET`

```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING
  lower(split_part(statement, ' ', 1)) = '\set' AND
  lower(split_part(statement, ' ', 2)) = 'quiet'
ORDER BY count DESC
LIMIT 10;
      statement      | count 
---------------------+-------
 \set QUIET 1        |   641
 \set QUIET 0        |    78
 \set QUIET OFF      |    42
 \set QUIET ON       |    42
 \set QUIET on       |    37
 \set QUIET yes      |    27
 \set QUIET off      |    26
 \set QUIET          |    18
 \set QUIET 1\r      |     3
 \set QUIET :QUIETRC |     2
(10 rows)
```

The `QUIET` variable has the same effect as the command-line option `-q` (`--quiet`). It configures psql to work silently without printing informational messages.

For example, normally `CREATE TABLE` outputs a confirmation text:
```sql
postgres=# CREATE TABLE t1 (i INT);
CREATE TABLE
postgres=# \set QUIET ON
postgres=# CREATE TABLE t2 (i INT);
postgres=# 
```
However, rather than using it interactively, it seems most users set `\set QUIET ON` at the **beginning** of their `psqlrc` and `\set QUIET OFF` at the **end** to suppress the output of the configuration commands running within the `psqlrc` itself.

#### `\set COMP_KEYWORD_CASE`

```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING
  lower(split_part(statement, ' ', 1)) = '\set' AND
  lower(split_part(statement, ' ', 2)) = 'comp_keyword_case'
ORDER BY count DESC
LIMIT 10;
               statement               | count 
---------------------------------------+-------
 \set COMP_KEYWORD_CASE upper          |   818
 \set COMP_KEYWORD_CASE lower          |    26
 \set COMP_KEYWORD_CASE preserve-upper |     5
 \set COMP_KEYWORD_CASE preserve-lower |     4
 \set COMP_KEYWORD_CASE upper\r        |     4
 \set COMP_KEYWORD_CASE 'upper'        |     2
 \set comp_keyword_case lower          |     1
(7 rows)
```

This variable determines whether to use upper or lower case when using tab completion for SQL keywords.

* `upper`: Use upper case.
* `lower`: Use lower case.
* `preserve-upper` (Default): Use the case of the already entered text; otherwise, use upper case.
* `preserve-lower`: Use the case of the already entered text; otherwise, use lower case.

The vast majority of users set this to `upper` to force uppercase auto-completion.

#### `\set HISTCONTROL`, `\set HISTFILE`, `\set HISTSIZE`

```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING
  lower(split_part(statement, ' ', 1)) = '\set' AND
  lower(split_part(statement, ' ', 2)) = 'histcontrol'
ORDER BY count DESC
LIMIT 10;
           statement           | count 
-------------------------------+-------
 \set HISTCONTROL ignoredups   |   748
 \set HISTCONTROL ignoreboth   |    60
 \set HISTCONTROL ignorespace  |    11
 \set HISTCONTROL ignoredups\r |     3
 \set HISTCONTROL none         |     1
(5 rows)
```

You can control the command history list using the `HISTCONTROL` variable.

- `ignorespace`: Lines starting with a space are not added to the history list.
- `ignoredups`: Lines matching the previous history entry are not added to the history list.
- `ignoreboth`: Enables both ignorespace and ignoredups.
- `none` (Default): All lines are added to the history list.

It seems many users configure `\set HISTCONTROL ignoredups` to prevent lines identical to the previous entry from being added to the history list.

```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING
  lower(split_part(statement, ' ', 1)) = '\set' AND
  lower(split_part(statement, ' ', 2)) = 'histfile'
ORDER BY count DESC
LIMIT 10;
                                                        statement                                                        | count 
-------------------------------------------------------------------------------------------------------------------------+-------
 \set HISTFILE ~/.psql_history- :DBNAME                                                                                  |   514
 \set HISTFILE `[[ -z $PSQL_HISTFILE ]] && echo $HOME/.psql_history || echo $PSQL_HISTFILE`                              |    46
 \set HISTFILE ~/.psql_history- :HOST - :DBNAME                                                                          |    39
 \set HISTFILE ~/.psql/history- :DBNAME                                                                                  |    26
 \set HISTFILE ~/psql_history- :DBNAME                                                                                   |    18
 \set HISTFILE ~/.psql_history-:DBNAME                                                                                   |    14
 \set HISTFILE ~/.history/psql- :HOST - :DBNAME                                                                          |    11
 \set HISTFILE <%= ENV['OPENSHIFT_DATA_DIR'] %>/.psql_history- :DBNAME                                                   |    10
 \set HISTFILE ~/.psql_history                                                                                           |     9
 \set HISTFILE ~/.psql_history/ :DBNAME                                                                                  |     8
(10 rows)
```

You can configure the filename for saving history using the `HISTFILE` variable.

It seems many users include the database name or hostname in the filename.

```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING
  lower(split_part(statement, ' ', 1)) = '\set' AND
  lower(split_part(statement, ' ', 2)) = 'histsize'
ORDER BY count DESC
LIMIT 10;
       statement       | count 
-----------------------+-------
 \set HISTSIZE 2000    |   109
 \set HISTSIZE 10000   |    31
 \set HISTSIZE 100000  |    30
 \set HISTSIZE 5000    |    23
 \set HISTSIZE 20000   |    13
 \set HISTSIZE 1000000 |    10
 \set HISTSIZE 1000    |     5
 \set HISTSIZE 12000   |     4
 \set HISTSIZE -1      |     4
 \set HISTSIZE 6000    |     4
(10 rows)
```

You can set the maximum number of commands to store in the command history (default is 500) using the `HISTSIZE` variable.

It seems many users configure this to increase the maximum number of commands stored in the history.

#### `\set VERBOSITY`
```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING
  lower(split_part(statement, ' ', 1)) = '\set' AND
  lower(split_part(statement, ' ', 2)) = 'verbosity'
ORDER BY count DESC
LIMIT 10;
         statement        | count 
--------------------------+-------
 \set VERBOSITY verbose   |   691
 \set VERBOSITY terse     |    15
 \set VERBOSITY default   |     6
 \set VERBOSITY verbose\r |     3
 \set VERBOSITY 'terse'   |     1
 \set VERBOSITY verbose;  |     1
(6 rows)
```
This controls the verbosity of error reports. The trend is overwhelmingly `verbose` (`\set VERBOSITY verbose`), ensuring error details are fully displayed.

#### `\set ON_ERROR_ROLLBACK`
```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING
  lower(split_part(statement, ' ', 1)) = '\set' AND
  lower(split_part(statement, ' ', 2)) = 'on_error_rollback'
ORDER BY count DESC
LIMIT 10;
                                                        statement                                                         | count 
--------------------------------------------------------------------------------------------------------------------------+-------
 \set ON_ERROR_ROLLBACK interactive                                                                                       |   227
 \set ON_ERROR_ROLLBACK on                                                                                                |    14
 \set ON_ERROR_ROLLBACK                                                                                                   |     4
 \set ON_ERROR_ROLLBACK 1                                                                                                 |     2
 \set ON_ERROR_ROLLBACK off                                                                                               |     2
 \set ON_ERROR_ROLLBACK 'interactive'                                                                                     |     2
 \set ON_ERROR_ROLLBACK 'on'\r                                                                                            |     1
 \set ON_ERROR_ROLLBACK 'on'                                                                                              |     1
 \set ON_ERROR_ROLLBACK off|on|interactive -- interactive lets you fix your fat fingers... Prefer to redo the whole thing |     1
(9 rows)
```
This controls behavior when an error occurs inside a transaction block.

* `on`: If a statement errors, the error is ignored, and the transaction continues.
* `interactive`: Errors are ignored only in interactive sessions.
* `off` (Default): An error aborts the entire transaction.

Most users set this to `interactive`, which allows you to "fix your fat fingers" without losing the whole transaction while typing manually.

#### `\set ON_ERROR_STOP`
```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING
  lower(split_part(statement, ' ', 1)) = '\set' AND
  lower(split_part(statement, ' ', 2)) = 'on_error_stop'
ORDER BY count DESC
LIMIT 10;
        statement        | count 
-------------------------+-------
 \set ON_ERROR_STOP on   |    69
 \set ON_ERROR_STOP      |     8
 \set ON_ERROR_STOP 1    |     5
 \set ON_ERROR_STOP true |     2
 \set ON_ERROR_STOP off  |     1
(5 rows)
```
By setting `\set ON_ERROR_STOP on`, command processing stops immediately after an error. In interactive mode, it returns to the prompt; in scripts, psql exits with error code 3.

---

### `\pset`

`\pset` controls the formatting of table output.

```sql
SELECT
  lower(split_part(statement, ' ', 1)) AS item1,
  lower(split_part(statement, ' ', 2)) AS item2,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY item1, item2            
HAVING lower(split_part(statement, ' ', 1)) = '\pset'
ORDER BY count DESC
LIMIT 30;
 item1 |          item2           | count 
-------+--------------------------+-------
 \pset | null                     |  1201
 \pset | linestyle                |   374
 \pset | border                   |   313
 \pset | pager                    |   289
 \pset | format                   |   144
 \pset | unicode_header_linestyle |    64
 \pset | unicode_column_linestyle |    57
 \pset | unicode_border_linestyle |    57
 \pset | expanded                 |    22
 \pset | numericlocale            |     9
 \pset | columns                  |     8
 \pset | footer                   |     8
 \pset | fieldsep                 |     6
 \pset | tuples_only              |     5
 \pset | pager_min_lines          |     3
 \pset | title                    |     3
 \pset | recordsep                |     2
(17 rows)
```

#### `\pset null`
```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING
  lower(split_part(statement, ' ', 1)) = '\pset' AND
  lower(split_part(statement, ' ', 2)) = 'null'
ORDER BY count DESC
LIMIT 10;
      statement      | count 
---------------------+-------
 \pset null '[NULL]' |   416
 \pset null '¤'      |   142
 \pset null ¤        |   127
 \pset null '(null)' |    98
 \pset null 'NULL'   |    93
 \pset null '[null]' |    51
 \pset null [null]   |    31
 \pset null NULL     |    28
 \pset null ∅        |    26
 \pset null '∅'      |    20
(10 rows)
```
Users configure this to display a specific string (like `[NULL]`, `(null)`, or `¤`) instead of a blank space when a value is `NULL`, making it easier to spot.

#### `\pset linestyle`
```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING
  lower(split_part(statement, ' ', 1)) = '\pset' AND
  lower(split_part(statement, ' ', 2)) = 'linestyle'
ORDER BY count DESC
LIMIT 10;
         statement         | count 
---------------------------+-------
 \pset linestyle unicode   |   320
 \pset linestyle 'unicode' |    44
 \pset linestyle ascii     |     5
 \pset linestyle u         |     2
 \pset linestyle unicode\r |     2
 \pset linestyle 'ascii'   |     1
(6 rows)
```
Many users set this to `unicode` (Default is `ascii`) to use smoother lines for tables.

**Example (Unicode):**
```text
           statement           │ count 
───────────────────────────────┼───────
 \pset linestyle unicode       │   320
 \pset linestyle 'unicode'     │    44
 \pset linestyle ascii         │     5
 \pset linestyle u             │     2
 \pset linestyle unicode\r     │     2
 \pset linestyle 'ascii'       │     1
(6 rows)
```

#### `\pset border`
```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING
  lower(split_part(statement, ' ', 1)) = '\pset' AND
  lower(split_part(statement, ' ', 2)) = 'border'
ORDER BY count DESC
LIMIT 10;
                     statement                      | count 
----------------------------------------------------+-------
 \pset border 2                                     |   250
 \pset border 1                                     |    48
 \pset border 0                                     |     7
 \pset border 3                                     |     3
 \pset border 2\r                                   |     2
 \pset border 1 -- Makes the borders a little nicer |     1
 \pset border 4                                     |     1
 \pset border 7                                     |     1
(8 rows)
```
This changes the border style of tables (Default is 1).

**Example (`border 0`):**
```text
                    statement                       count 
-------------------------------------------------- -----
\pset border 2                                       250
\pset border 1                                        48
\pset border 0                                         7
\pset border 3                                         3
\pset border 2\r                                       2
\pset border 1 -- Makes the borders a little nicer     1
\pset border 4                                         1
\pset border 7                                         1
(8 rows)
```

**Example (`border 2`):**
```text
+----------------------------------------------------+-------+
|                    statement                       | count |
+----------------------------------------------------+-------+
| \pset border 2                                     |   250 |
| \pset border 1                                     |    48 |
| \pset border 0                                     |     7 |
| \pset border 3                                     |     3 |
| \pset border 2\r                                   |     2 |
| \pset border 1 -- Makes the borders a little nicer |     1 |
| \pset border 4                                     |     1 |
| \pset border 7                                     |     1 |
+----------------------------------------------------+-------+
(8 rows)
```

#### `\pset pager`
```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING
  lower(split_part(statement, ' ', 1)) = '\pset' AND
  lower(split_part(statement, ' ', 2)) = 'pager'
ORDER BY count DESC
LIMIT 10;
      statement       | count 
----------------------+-------
 \pset pager off      |   181
 \pset pager always   |    69
 \pset pager on       |    28
 \pset pager          |     4
 \pset pager on\r     |     2
 \pset pager auto     |     2
 \pset pager always\r |     1
 \pset pager 1        |     1
 \pset pager 0        |     1
(9 rows)
```
* `off`: Disable the pager.
* `always`: Always use the pager.
* `on` (Default): Use pager only when needed.

#### `\pset format`
```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING
  lower(split_part(statement, ' ', 1)) = '\pset' AND
  lower(split_part(statement, ' ', 2)) = 'format'
ORDER BY count DESC
LIMIT 10;
        statement        | count 
------------------------+-------
 \pset format wrapped   |   127
 \pset format aligned   |    12
 \pset format           |     2
 \pset format csv       |     2
 \pset format unaligned |     1
(5 rows)
```
Setting `wrapped` helps display wide data values by wrapping them across multiple lines to fit the target column width. (Default is `aligned`).

---

### `\echo`

```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING lower(split_part(statement, ' ', 1)) = '\echo'
ORDER BY count DESC
LIMIT 30;
                              statement                               | count 
----------------------------------------------------------------------+-------
 \echo 'Administrative queries:\n'                                    |    35
 \echo '\nCurrent Host Server Date Time : '`date` '\n'                |    33
 \echo                                                                |    29
 \echo 'Development queries:\n'                                       |    28
 \echo '\t\t\t\\h\t\t-- Help with SQL commands'                       |    20
 \echo '\t\t\t:uselesscol\t-- Useless columns'                        |    20
 \echo '\t\t\t:activity\t-- Server activity'                          |    20
 \echo '\t\t\t:conninfo\t-- Server connections'                       |    20
 \echo '\t\t\t:uptime\t\t-- Server uptime'                            |    20
 \echo 'Type :extensions to see the available extensions. \n'          |    20
 \echo '\t\t\t:locks\t\t-- Lock info'                                 |    20
 \echo '\t\t\t:clear\t\t-- Clear screen'                              |    19
 \echo '\t\t\t:tablesize\t-- Tables Size'                             |    19
 \echo 'Type :version to see the PostgreSQL version. \n'              |    19
 \echo '\t\t\t:menu\t\t-- Help Menu'                                  |    19
 \echo 'Type \\q to exit. \n'                                         |    19
 \echo '\t\t\t\\?\t\t-- Help with psql commands\n'                    |    19
 \echo '\t\t\t:sp\t\t-- Current Search Path'                          |    19
 \echo '\t\t\t:dbsize\t\t-- Database Size'                            |    19
 \echo '\t\t\t:ll\t\t-- List\n'                                       |    19
 \echo '\t\t\t:settings\t-- Server Settings'                          |    19
 \echo '\t\t\t:waits\t\t-- Waiting queires'                           |    18
 \echo 'Welcome to PostgreSQL! \n'                                    |    15
 \echo '\t:activity\t-- Server activity'                              |    11
 \echo '\t:dbsize\t\t-- Database Size'                                |    11
 \echo '\t:tablesize\t-- Tables Size'                                 |    11
 \echo '\t:uptime\t\t-- Server uptime'                                |    11
 \echo '\t:conninfo\t-- Server connections'                           |    11
 \echo '\t:settings\t-- Server Settings'                              |    11
 \echo '\t\\?\t\t-- Help with psql commands\n'                        |    10
(30 rows)
```
`\echo` is used to print messages.
Looking at the data, users utilize this to display "Welcome" messages, server information (Host, Date), or cheat sheets for custom aliases upon psql startup.

---

### `\x`

```sql
SELECT
  lower(split_part(statement, ' ', 1)) AS item1,
  lower(split_part(statement, ' ', 2)) AS item2,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY item1, item2            
HAVING lower(split_part(statement, ' ', 1)) = '\x'
ORDER BY count DESC
LIMIT 30;
 item1 | item2  | count 
-------+--------+-------
 \x    | auto   |   987
 \x    |        |    11
 \x    | off    |    10
 \x    | on     |     9
 \x    | auto\r |     1
(5 rows)
```

`\x auto` is the most common setting (987 uses). This automatically switches to extended table formatting (where columns are listed vertically) when the output is too wide to fit the screen.

---

### `\timing`

```sql
SELECT
  lower(split_part(statement, ' ', 1)) AS item1,
  lower(split_part(statement, ' ', 2)) AS item2,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY item1, item2            
HAVING lower(split_part(statement, ' ', 1)) = '\timing'
ORDER BY count DESC
LIMIT 30;
  item1  | item2 | count 
---------+-------+-------
 \timing |       |   833
 \timing | on    |   159
 \timing | off   |     2
 \timing | on\r  |     1
(4 rows)
```

Using `\timing` or `\timing on` toggles the display of execution time for every SQL statement.

---

### `\unset`

```sql
SELECT
  lower(split_part(statement, ' ', 1)) AS item1,
  lower(split_part(statement, ' ', 2)) AS item2,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY item1, item2            
HAVING lower(split_part(statement, ' ', 1)) = '\unset'
ORDER BY count DESC
LIMIT 30;
 item1  |   item2    | count 
--------+------------+-------
 \unset | quiet      |   554
 \unset | timing     |     3
 \unset | quiet\r    |     3
 \unset | quietrc    |     2
 \unset | singlestep |     2
 \unset | quite      |     1
(6 rows)
```

As mentioned in the `\set QUIET` section, `\unset` is primarily used to `\unset QUIET` at the end of the file to restore message output after loading the configuration.

---

### `\setenv`

```sql
SELECT
  lower(split_part(statement, ' ', 1)) AS item1,
  lower(split_part(statement, ' ', 2)) AS item2,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY item1, item2            
HAVING lower(split_part(statement, ' ', 1)) = '\setenv'
ORDER BY count DESC
LIMIT 30;
  item1  |            item2             | count 
---------+----------------------------+-------
 \setenv | pager                      |   129
 \setenv | less                       |    89
 \setenv | editor                     |    41
 \setenv | psql_editor                |     9
 \setenv | psql_pager                 |     2
 \setenv | psql_editor_linenumber_arg |     2
 \setenv | echo                       |     1
(7 rows)
```

```sql
SELECT              
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING 
  lower(split_part(statement, ' ', 1)) = '\setenv' AND
  lower(split_part(statement, ' ', 2)) = 'pager'
ORDER BY count DESC
LIMIT 10;
                               statement                               | count 
-----------------------------------------------------------------------+-------
 \setenv PAGER 'less -SX'                                              |    28
 \setenv PAGER less                                                    |    26
 \setenv PAGER pspg                                                    |    14
 \setenv PAGER 'less -S'                                               |    11
 \setenv PAGER 'pspg --no-mouse -bX --no-commandbar --no-topbar'       |     5
 \setenv PAGER 'less'                                                  |     4
 \setenv PAGER 'less -XS'                                              |     4
 \setenv PAGER 'pspg -FX -s 17 --no-mouse --no-bars --only-for-tables' |     2
 \setenv PAGER 'less -FXE'                                             |     2
 \setenv PAGER '/usr/bin/less'                                         |     2
(10 rows)
```

It seems users use less or pspg as their pager by configuring the PAGER or PSQL_PAGER environment variables. Also, it seems they configure options for less by setting the LESS environment variable.

{% details 📝 **Note:** %}
pspg is a pager specifically designed for PostgreSQL tables.
https://github.com/okbob/pspg
{% enddetails %}

```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING 
  lower(split_part(statement, ' ', 1)) = '\setenv' AND
  lower(split_part(statement, ' ', 2)) = 'editor'
ORDER BY count DESC
LIMIT 10;
                 statement                | count 
------------------------------------------+-------
 \setenv EDITOR 'nvim'                    |     5
 \setenv EDITOR 'vim'                     |     5
 \setenv EDITOR '/usr/bin/vim'            |     5
 \setenv EDITOR '/usr/local/bin/vim'      |     4
 \setenv EDITOR vim                       |     3
 \setenv EDITOR '/usr/bin/nvim'           |     2
 \setenv EDITOR emacs                     |     2
 \setenv EDITOR nvim                      |     2
 \setenv EDITOR '~/.nix-profile/bin/nvim' |     2
 \setenv EDITOR /usr/bin/vim              |     1
(10 rows)
```

It seems users use Neovim, Vim, or Emacs as their editor by configuring the EDITOR or PSQL_EDITOR environment variables.

---

### `\r`
This stands for `\reset`, which clears the query buffer. In my data, it looks mostly like a parsing artifact where `\r` (carriage return) characters were caught, but some users might be ensuring a clean slate.

---

### `SET` (SQL Command)

```sql
SELECT
  lower(split_part(statement, ' ', 1)) AS item1,
  lower(split_part(statement, ' ', 2)) AS item2,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY item1, item2            
HAVING lower(split_part(statement, ' ', 1)) = 'set'
ORDER BY count DESC
LIMIT 30;
 item1 |                          item2                          | count 
-------+--------------------------------------------------------+-------
 set   | search_path                                            |    47
 set   | intervalstyle                                          |    41
 set   | timezone                                               |     9
 set   | application_name                                       |     7
 set   | client_min_messages                                    |     6
 set   | bytea_output                                           |     5
 set   | client_encoding                                        |     4
 set   | search_path=public;                                    |     1
 set   | session                                                |     1
 set   | search_path=site_config_service,project_service,public |     1
 set   | work_mem                                               |     1
 set   | timezone='us/eastern';                                 |     1
 set   | default_transaction_read_only                          |     1
 set   | search_path=dwh,mart,stage                             |     1
 set   | tcp_keepalives_idle                                    |     1
 set   | maintenance_work_mem='1gb';                            |     1
 set   | search_path=public,postgis                             |     1
 set   | work_mem='512mb';                                      |     1
 set   | enable_bitmapscan                                      |     1
 set   | statement_timeout                                      |     1
(20 rows)
```

```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING 
  lower(split_part(statement, ' ', 1)) = 'set' AND
  lower(split_part(statement, ' ', 2)) = 'search_path'
ORDER BY count DESC
LIMIT 10;
                        statement                        | count 
--------------------------------------------------------+-------
 set search_path to pdfbox,mycore,public;               |     3
 set search_path to setcore,public;                     |     3
 set search_path to curl7,mycore,public;                |     3
 set search_path to jsonorg,setcore,public;             |     3
 set search_path to mycore,setcore,public;              |     3
 set search_path to public,setcore,libxml2;             |     3
 SET search_path TO setcore,public;                     |     3
 SET search_path TO expat2,mycore,setcore,public;       |     3
 set search_path to gnuzip,setcore,public;              |     3
 SET search_path TO arxiv,pdfbox,mycore,setcore,public; |     2
(10 rows)
```
```sql
SELECT
  statement,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY lower(split_part(statement, ' ', 1)), statement            
HAVING 
  lower(split_part(statement, ' ', 1)) = 'set' AND
  lower(split_part(statement, ' ', 2)) = 'intervalstyle'
ORDER BY count DESC
LIMIT 10;
                 statement                | count 
------------------------------------------+-------
 set intervalstyle to 'postgres_verbose'; |    39
 SET intervalstyle to 'postgres_verbose'; |     1
 set intervalstyle to 'postgres_verbose'  |     1
(3 rows)
```

These are standard SQL commands executed at startup.

* `search_path`: Users set custom search paths for their schemas.
* `intervalstyle`: Users often set this to `'postgres_verbose'` to control how interval data types are formatted.

---

### `\encoding`
```sql
SELECT
  lower(split_part(statement, ' ', 1)) AS item1,
  lower(split_part(statement, ' ', 2)) AS item2,
  COUNT(statement) AS count
FROM psqlrc
GROUP BY item1, item2            
HAVING lower(split_part(statement, ' ', 1)) = '\encoding'
ORDER BY count DESC
LIMIT 30;
   item1   |   item2   | count 
-----------+-----------+-------
 \encoding | unicode   |   105
 \encoding | utf8      |     6
 \encoding | latin1    |     5
 \encoding | utf-8     |     2
 \encoding | sql_ascii |     2
 \encoding | 'utf8'    |     1
 \encoding | latin1\r  |     1
 \encoding |           |     1
(8 rows)
```

Most users explicitly set the client encoding to `unicode`.

---

## Conclusion

I had never customized my `psqlrc` before, so this investigation was a great opportunity to learn about the available settings.

Personally, since I often connect to multiple different PostgreSQL servers, I plan to customize my **PROMPT** settings to clearly distinguish between environments (and avoid accidents!).

I hope this article helps you create a more comfortable `psql` life.