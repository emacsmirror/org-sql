# Org-SQL ![Github Workflow Status](https://img.shields.io/github/workflow/status/ndwarshuis/org-sql/CI) ![MELPA VERSION](https://melpa.org/packages/org-sql-badge.svg)

This is a SQL backend for Emacs Org-Mode. It scans through text files formatted
in org-mode, parses them, and adds key information such as todo keywords,
timestamps, and links to a relational database. For now only SQLite is
supported.

# Motivation and Goals

Despite the fact that Emacs is the (second?) greatest text editor of all time,
it is not a good data analysis platform, and neither is Lisp a good data
analysis language. This is the strong-suite of other languages such as Python
and R which have powerful data manipulation packages (`dplyr` and `pandas`) as
well as specialized presentation platforms (`shiny` and `dash`). The common
thread between these data analysis tools is a tabular data storage system, for
which SQL is the most common language.

Therefore, the primary goal of `org-sql` is to provide a link between the
text-based universe of Emacs / Org-mode and the table-based universe of data
analysis platforms.

A common use case for org-mode is a daily planner. Within this use case, some
questions that can easily be answered with a SQL-backed approach:
- How much time do I spend doing pre-planned work? (track how much time is spent
  clocking)
- How well do I estimate how long tasks take? (compare effort and clocked time)
- Which types of tasks do I concentrate on the most? (tag entries and sort based
  on effort and clocking)
- How indecisive am I? (track how many times schedule or deadline timestamps are
  changed)
- How much do I overplan? (track number of canceled tasks)
- How much do I delegate (track properties indicating which people are working
  on tasks)
- How many outstanding tasks/projects do I have? (count keywords and tags on
  headlines)

There are other uses for an Org-mode SQL database. If one has many org files
scattered throughout their filesystem, a database is an easy way to aggregate
key information such as links or timestamps. Or if one primary uses org-mode for
taking notes, it could be a way to aggregate and analyze meeting minutes.

Of course, these could all be done directly in Org-mode with Lisp code (indeed
there are already built-in functions for reporting aggregated effort and
clock-time). But why do that when one could analyze all org files by making a
descriptive dashboard with a relatively few lines of R or Python code?

# Installation

Download the package from MELPA

```
M-x package-install RET org-sql RET
```

Alternatively, clone this repository into your config directory

``` sh
git clone git@github.com:ndwarshuis/org-sql.git ~/config/path/org-sql/
```

Once obtained, add the package to `load-path` and require it

``` emacs-lisp
(add-to-list 'load-path "~/config/path/org-sql/")
(require 'org-sql)
```

One can also use `use-package` to automate this entire process

``` emacs-lisp
(use-package org-sql
  :ensure t
  :config
  ;; add config options here...
  )
```

# Configuration

## General Behavior

- `org-sql-db-config`: a list describing the database to use and how to connect
  to it (see this variable's help page for more details)
- `org-sql-files`: list of org files to insert into database
- `org-sql-debug`: turn on SQL transaction debug output in the message buffer

## Database Storage

Options following the pattern `org-sql-exclude-X` dictate what to exclude from
the database. By default all these variables are nil (include everything).

## Logbooks

Much of the extracted data from `org-sql` pertains to logbook entries, and there
are a number of settings that effect how this data is generated in org files and
how it may be parsed reliably.

Firstly, one needs to set the relevant `org-mode` variables in order to capture
logging information. Please refer to the documentation in `org-mode` itself for
their meaning:
- `org-log-done`
- `org-log-reschedule`
- `org-log-redeadline`
- `org-log-note-clock-out`
- `org-log-refile`
- `org-log-repeat`
- `org-todo-keywords` (in this one can set which todo keywords changes are
  logged)

Obtaining the above information for the database assumes that
`org-log-note-headings` is left at its default value. This limitation may be
surpassed in the future.

Additionally, for best results it is recommended that all logbook entries be
contained in their own drawer. This means that `org-log-into-drawer` should be
set to `LOGBOOK` and `org-clock-into-drawer` should be set to `t` (which means
clocks go into a drawer with hardcoded name `LOGBOOK`). Without these settings,
`org-sql` needs to guess where the logbook entries are based on location and
pattern matching, which is not totally reliable.

# Usage

## Initializing

Run `org-sql-user-reset`. This will create a new database and initialize it with
the default schema. It will also delete an existing database before creating the
new one if it exists.

## Updating

Run `org-sql-user-update`. This will synchronize the database with all files as
indicated in `org-sql-files` by first checking if the file is in the database
and inserting it if not. If the file is already present, it will check the md5
to assess if updates are needed. Note that for any file in the database,
changing even one character in the file on disk will trigger an deletion of the
file in the database followed by an insertion of the *entire* org file.

This may take several seconds/minutes if inserting many files depending on the
speed of your device (particularly IO) and the size/number of files. This
operation will also block Emacs until complete.

## Removing all data

Run `org-sql-user-clear-all`. This will clear all data but leave the schema.

# Database Layout

## General design features

- the schema is identical except for types between all database implementations
  (SQLite, Postgres, etc)
- all foreign keys are set with `UPDATE CASCADE` and `DELETE CASCADE`
- all time values are stores as unixtime (integers in seconds)

## Schema

### files

Each row stores metadata for one tracked org file

| Column | Is Primary | Foreign Keys (parent - table) | NULL Allowed | Type (SQLite / Postgres) | Description |
|  -  |  -  |  -  |  -  |  -  |  -  |
| file_path | x |  |  | TEXT / TEXT | path to the org file |
| md5 |  |  |  | TEXT / TEXT | md5 checksum of the org file |
| size |  |  |  | INTEGER / INTEGER | size of the org file in bytes |

### headlines

Each row stores one headline in a given org file and its metadata

| Column | Is Primary | Foreign Keys (parent - table) | NULL Allowed | Type (SQLite / Postgres) | Description |
|  -  |  -  |  -  |  -  |  -  |  -  |
| file_path | x | file_path - files |  | TEXT / TEXT | path to file containin the headline |
| headline_offset | x |  |  | INTEGER / INTEGER | file offset of the headline's first character |
| headline_text |  |  |  | TEXT / TEXT | raw text of the headline |
| keyword |  |  | x | TEXT / TEXT | the TODO state keyword |
| effort |  |  | x | INTEGER / INTEGER | the value of the Effort property in minutes |
| priority |  |  | x | TEXT / TEXT | character value of the priority |
| is_archived |  |  |  | INTEGER / BOOLEAN | true if the headline has an archive tag |
| is_commented |  |  |  | INTEGER / BOOLEAN | true if the headline has a comment keyword |
| content |  |  | x | TEXT / TEXT | the headline contents |

### headline_closures

Each row stores the ancestor and depth of a headline relationship (eg closure table)

| Column | Is Primary | Foreign Keys (parent - table) | NULL Allowed | Type (SQLite / Postgres) | Description |
|  -  |  -  |  -  |  -  |  -  |  -  |
| file_path | x | file_path - headlines, file_path - headlines |  | TEXT / TEXT | path to the file containing this headline |
| headline_offset | x | headline_offset - headlines |  | INTEGER / INTEGER | offset of this headline |
| parent_offset | x | headline_offset - headlines |  | INTEGER / INTEGER | offset of this headline's parent |
| depth |  |  | x | INTEGER / INTEGER | levels between this headline and the referred parent |

### timestamps

Each row stores one timestamp

| Column | Is Primary | Foreign Keys (parent - table) | NULL Allowed | Type (SQLite / Postgres) | Description |
|  -  |  -  |  -  |  -  |  -  |  -  |
| file_path | x | file_path - headlines |  | TEXT / TEXT | path to the file containing this timestamp |
| headline_offset |  | headline_offset - headlines |  | INTEGER / INTEGER | offset of the headline containing this timestamp |
| timestamp_offset | x |  |  | INTEGER / INTEGER | offset of this timestamp |
| raw_value |  |  |  | TEXT / TEXT | text representation of this timestamp |
| is_active |  |  |  | INTEGER / BOOLEAN | true if the timestamp is active |
| warning_type |  |  | x | TEXT / ENUM | warning type of this timestamp (`all`, or `first`) |
| warning_value |  |  | x | INTEGER / INTEGER | warning shift of this timestamp |
| warning_unit |  |  | x | TEXT / ENUM | warning unit of this timestamp  (`hour`, `day`, `week`, `month`, or `year`) |
| repeat_type |  |  | x | TEXT / ENUM | repeater type of this timestamp (`catch-up`, `restart`, or `cumulate`) |
| repeat_value |  |  | x | INTEGER / INTEGER | repeater shift of this timestamp |
| repeat_unit |  |  | x | TEXT / ENUM | repeater unit of this timestamp (`hour`, `day`, `week`, `month`, or `year`) |
| time_start |  |  |  | INTEGER / INTEGER | the start time (or only time) of this timestamp |
| time_end |  |  | x | INTEGER / INTEGER | the end time of this timestamp |
| start_is_long |  |  |  | INTEGER / BOOLEAN | true if the start time is in long format |
| end_is_long |  |  | x | INTEGER / BOOLEAN | true if the end time is in long format |

### planning_entries

Each row stores the metadata for headline planning timestamps.

| Column | Is Primary | Foreign Keys (parent - table) | NULL Allowed | Type (SQLite / Postgres) | Description |
|  -  |  -  |  -  |  -  |  -  |  -  |
| file_path | x | file_path - timestamps |  | TEXT / TEXT | path to the file containing the entry |
| headline_offset | x |  |  | INTEGER / INTEGER | file offset of the headline with this tag |
| planning_type | x |  |  | TEXT / ENUM | the type of this planning entry (`closed`, `scheduled`, or `deadline`) |
| timestamp_offset |  | timestamp_offset - timestamps |  | INTEGER / INTEGER | file offset of this entries timestamp |

### file_tags

Each row stores one tag at the file level

| Column | Is Primary | Foreign Keys (parent - table) | NULL Allowed | Type (SQLite / Postgres) | Description |
|  -  |  -  |  -  |  -  |  -  |  -  |
| file_path | x | file_path - files |  | TEXT / TEXT | path to the file containing the tag |
| tag | x |  |  | TEXT / TEXT | the text value of this tag |

### headline_tags

Each row stores one tag

| Column | Is Primary | Foreign Keys (parent - table) | NULL Allowed | Type (SQLite / Postgres) | Description |
|  -  |  -  |  -  |  -  |  -  |  -  |
| file_path | x | file_path - headlines |  | TEXT / TEXT | path to the file containing the tag |
| headline_offset | x | headline_offset - headlines |  | INTEGER / INTEGER | file offset of the headline with this tag |
| tag | x |  |  | TEXT / TEXT | the text value of this tag |
| is_inherited | x |  |  | INTEGER / BOOLEAN | true if this tag is from the ITAGS property |

### properties

Each row stores one property

| Column | Is Primary | Foreign Keys (parent - table) | NULL Allowed | Type (SQLite / Postgres) | Description |
|  -  |  -  |  -  |  -  |  -  |  -  |
| file_path | x | file_path - files |  | TEXT / TEXT | path to the file containing this property |
| property_offset | x |  |  | INTEGER / INTEGER | file offset of this property in the org file |
| key_text |  |  |  | TEXT / TEXT | this property's key |
| val_text |  |  |  | TEXT / TEXT | this property's value |

### file_properties

Each row stores a property at the file level

| Column | Is Primary | Foreign Keys (parent - table) | NULL Allowed | Type (SQLite / Postgres) | Description |
|  -  |  -  |  -  |  -  |  -  |  -  |
| file_path | x | file_path - files, file_path - properties |  | TEXT / TEXT | path to file containin the property |
| property_offset | x | property_offset - properties |  | INTEGER / INTEGER | file offset of this property in the org file |

### headline_properties

Each row stores a property at the headline level

| Column | Is Primary | Foreign Keys (parent - table) | NULL Allowed | Type (SQLite / Postgres) | Description |
|  -  |  -  |  -  |  -  |  -  |  -  |
| file_path | x | file_path - headlines |  | TEXT / TEXT | path to file containin the property |
| property_offset | x |  |  | INTEGER / INTEGER | file offset of this property in the org file |
| headline_offset |  | headline_offset - headlines |  | INTEGER / INTEGER | file offset of the headline with this property |

### clocks

Each row stores one clock entry

| Column | Is Primary | Foreign Keys (parent - table) | NULL Allowed | Type (SQLite / Postgres) | Description |
|  -  |  -  |  -  |  -  |  -  |  -  |
| file_path | x | file_path - headlines |  | TEXT / TEXT | path to the file containing this clock |
| headline_offset |  | headline_offset - headlines |  | INTEGER / INTEGER | offset of the headline with this clock |
| clock_offset | x |  |  | INTEGER / INTEGER | file offset of this clock |
| time_start |  |  | x | INTEGER / INTEGER | timestamp for the start of this clock |
| time_end |  |  | x | INTEGER / INTEGER | timestamp for the end of this clock |
| clock_note |  |  | x | TEXT / TEXT | the note entry beneath this clock |

### logbook_entries

Each row stores one logbook entry (except for clocks)

| Column | Is Primary | Foreign Keys (parent - table) | NULL Allowed | Type (SQLite / Postgres) | Description |
|  -  |  -  |  -  |  -  |  -  |  -  |
| file_path | x | file_path - headlines |  | TEXT / TEXT | path to the file containing this entry |
| headline_offset |  | headline_offset - headlines |  | INTEGER / INTEGER | offset of the headline with this entry |
| entry_offset | x |  |  | INTEGER / INTEGER | offset of this logbook entry |
| entry_type |  |  | x | TEXT / TEXT | type of this entry (see `org-log-note-headlines`) |
| time_logged |  |  | x | INTEGER / INTEGER | timestamp for when this entry was taken |
| header |  |  | x | TEXT / TEXT | the first line of this entry (usually standardized) |
| note |  |  | x | TEXT / TEXT | the text of this entry underneath the header |

### state_changes

Each row stores additional metadata for a state change logbook entry

| Column | Is Primary | Foreign Keys (parent - table) | NULL Allowed | Type (SQLite / Postgres) | Description |
|  -  |  -  |  -  |  -  |  -  |  -  |
| file_path | x | file_path - logbook_entries |  | TEXT / TEXT | path to the file containing this entry |
| entry_offset | x | entry_offset - logbook_entries |  | INTEGER / INTEGER | offset of the logbook entry for this state change |
| state_old |  |  |  | TEXT / TEXT | former todo state keyword |
| state_new |  |  |  | TEXT / TEXT | updated todo state keyword |

### planning_changes

Each row stores additional metadata for a planning change logbook entry

| Column | Is Primary | Foreign Keys (parent - table) | NULL Allowed | Type (SQLite / Postgres) | Description |
|  -  |  -  |  -  |  -  |  -  |  -  |
| file_path | x | file_path - timestamps, file_path - logbook_entries |  | TEXT / TEXT | path to the file containing this entry |
| entry_offset | x | entry_offset - logbook_entries |  | INTEGER / INTEGER | offset of the logbook entry for this planning change |
| timestamp_offset |  | timestamp_offset - timestamps |  | INTEGER / INTEGER | offset of the former timestamp |

### links

Each rows stores one link

| Column | Is Primary | Foreign Keys (parent - table) | NULL Allowed | Type (SQLite / Postgres) | Description |
|  -  |  -  |  -  |  -  |  -  |  -  |
| file_path | x | file_path - headlines |  | TEXT / TEXT | path to the file containing this link |
| headline_offset |  | headline_offset - headlines |  | INTEGER / INTEGER | offset of the headline with this link |
| link_offset | x |  |  | INTEGER / INTEGER | file offset of this link |
| link_path |  |  |  | TEXT / TEXT | target of this link (eg url, file path, etc) |
| link_text |  |  | x | TEXT / TEXT | text of this link |
| link_type |  |  |  | TEXT / TEXT | type of this link (eg http, mu4e, file, etc) |

<!-- 0.0.1 -->

# Changelog

## 1.0.0 (relative to previous unversioned release)

- use `org-ml` to simplify code
- use shell commands and temp scripts for SQL interaction instead of built-in
  Emacs SQL comint mode
- various performance improvements
- make entries for `closed`, `scheduled`, and `deadline` in `headlines` table
  and remove the `planning_type` entry in `timestamp` table

# Acknowledgements

The idea for this is based on ![John Kitchin's](http://kitchingroup.cheme.cmu.edu/blog/2017/01/03/Find-stuff-in-org-mode-anywhere/)
implementation, which uses `emacsql` as the SQL backend.
