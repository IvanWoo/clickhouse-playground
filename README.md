# clickhouse-playground <!-- omit in toc -->

- [setup](#setup)
- [crazy stuff](#crazy-stuff)
  - [git-import](#git-import)
    - [prepare the table schema](#prepare-the-table-schema)
    - [generate and insert the data](#generate-and-insert-the-data)
    - [explore the datasets](#explore-the-datasets)
- [ref](#ref)

## setup

install required binaries via brew

```sh
brew install vagrant
brew install qemu
```

install the provider for vagrant

```sh
vagrant plugin install vagrant-qemu
```

access the machine

```sh
vagrant up
vagrant ssh
```

or append the ssh info into ~/.ssh/config

```sh
vagrant ssh-config >> ~/.ssh/config
ssh clickhouse
```

halt the machine

```sh
vagrant halt
```

reset the machine

```sh
vagrant destroy
```

## crazy stuff

start a clickhouse instance

```sh
tmux
clickhouse server
```

`CTRL+b d` to detach the tmux session

### git-import

#### prepare the table schema

save below sql into `git.sql`

```sql
DROP DATABASE IF EXISTS git;
CREATE DATABASE git;
CREATE TABLE git.commits
(
    hash String,
    author LowCardinality(String),
    time DateTime,
    message String,
    files_added UInt32,
    files_deleted UInt32,
    files_renamed UInt32,
    files_modified UInt32,
    lines_added UInt32,
    lines_deleted UInt32,
    hunks_added UInt32,
    hunks_removed UInt32,
    hunks_changed UInt32
) ENGINE = MergeTree ORDER BY time;
CREATE TABLE git.file_changes
(
    change_type Enum('Add' = 1, 'Delete' = 2, 'Modify' = 3, 'Rename' = 4, 'Copy' = 5, 'Type' = 6),
    path LowCardinality(String),
    old_path LowCardinality(String),
    file_extension LowCardinality(String),
    lines_added UInt32,
    lines_deleted UInt32,
    hunks_added UInt32,
    hunks_removed UInt32,
    hunks_changed UInt32,
    commit_hash String,
    author LowCardinality(String),
    time DateTime,
    commit_message String,
    commit_files_added UInt32,
    commit_files_deleted UInt32,
    commit_files_renamed UInt32,
    commit_files_modified UInt32,
    commit_lines_added UInt32,
    commit_lines_deleted UInt32,
    commit_hunks_added UInt32,
    commit_hunks_removed UInt32,
    commit_hunks_changed UInt32
) ENGINE = MergeTree ORDER BY time;
CREATE TABLE git.line_changes
(
    sign Int8,
    line_number_old UInt32,
    line_number_new UInt32,
    hunk_num UInt32,
    hunk_start_line_number_old UInt32,
    hunk_start_line_number_new UInt32,
    hunk_lines_added UInt32,
    hunk_lines_deleted UInt32,
    hunk_context LowCardinality(String),
    line LowCardinality(String),
    indent UInt8,
    line_type Enum('Empty' = 0, 'Comment' = 1, 'Punct' = 2, 'Code' = 3),
    prev_commit_hash String,
    prev_author LowCardinality(String),
    prev_time DateTime,
    file_change_type Enum('Add' = 1, 'Delete' = 2, 'Modify' = 3, 'Rename' = 4, 'Copy' = 5, 'Type' = 6),
    path LowCardinality(String),
    old_path LowCardinality(String),
    file_extension LowCardinality(String),
    file_lines_added UInt32,
    file_lines_deleted UInt32,
    file_hunks_added UInt32,
    file_hunks_removed UInt32,
    file_hunks_changed UInt32,
    commit_hash String,
    author LowCardinality(String),
    time DateTime,
    commit_message String,
    commit_files_added UInt32,
    commit_files_deleted UInt32,
    commit_files_renamed UInt32,
    commit_files_modified UInt32,
    commit_lines_added UInt32,
    commit_lines_deleted UInt32,
    commit_hunks_added UInt32,
    commit_hunks_removed UInt32,
    commit_hunks_changed UInt32
) ENGINE = MergeTree ORDER BY time;
```

```sh
clickhouse client --multiquery < git.sql
```

#### generate and insert the data

```sh
git clone https://github.com/thi-ng/umbrella.git
cd umbrella
clickhouse git-import
```

```sh
$ wc -lc commits.tsv file_changes.tsv line_changes.tsv
     7542   1058859 commits.tsv
   107905  20650215 file_changes.tsv
  1698696 580062827 line_changes.tsv
  1814143 601771901 total
```

```sh
clickhouse client --query "INSERT INTO git.commits FORMAT TSV" < commits.tsv
clickhouse client --query "INSERT INTO git.file_changes FORMAT TSV" < file_changes.tsv
clickhouse client --query "INSERT INTO git.line_changes FORMAT TSV" < line_changes.tsv
```

#### explore the datasets

```sh
clickhosue client
use git
```

the matrix of authors: who rewrites another author's code

```sql
SELECT prev_author, author AS new_author, count() AS c
FROM line_changes WHERE sign = -1 AND file_extension IN ('ts')
AND line_type NOT IN ('Punct', 'Empty')
AND author != prev_author AND prev_author != ''
GROUP BY prev_author, author ORDER BY c DESC
LIMIT 1 BY prev_author LIMIT 100
```

```sh
┌─prev_author──────┬─new_author──────┬───c─┐
│ Bnaya Peretz     │ Karsten Schmidt │ 219 │
│ Alberto Massa    │ Karsten Schmidt │ 125 │
│ Karsten Schmidt  │ Bnaya Peretz    │ 121 │
│ Gavin Cannizzaro │ Karsten Schmidt │ 116 │
│ Matei Adriel     │ Karsten Schmidt │  68 │
│ André Wachter    │ Karsten Schmidt │  56 │
│ alberto          │ Karsten Schmidt │  42 │
│ Alberto          │ Karsten Schmidt │  35 │
│ Jamie Owen       │ Karsten Schmidt │  17 │
│ Arthur Carabott  │ Karsten Schmidt │  16 │
│ David Negstad    │ Karsten Schmidt │  12 │
│ evilive          │ Karsten Schmidt │  12 │
│ d3v53c           │ Karsten Schmidt │  10 │
│ stwind           │ Karsten Schmidt │   6 │
│ arcticnoah       │ Karsten Schmidt │   2 │
│ Jamie Slome      │ Karsten Schmidt │   2 │
│ Pierre Grimaud   │ Karsten Schmidt │   1 │
└──────────────────┴─────────────────┴─────┘
```

order files by the number of authors

```sql
SELECT path, count(DISTINCT author) as c_author
FROM file_changes
WHERE file_extension IN ('ts')
GROUP BY path
ORDER BY c_author DESC
LIMIT 20;
```

```sh
┌─path───────────────────────────────────────────┬─c_author─┐
│ packages/checks/src/is-prototype-polluted.ts   │        3 │
│ packages/rstream-gestures/src/index.ts         │        3 │
│ packages/checks/src/index.ts                   │        3 │
│ packages/transducers/src/xform/flatten-with.ts │        2 │
│ packages/transducers/src/xform/match-last.ts   │        2 │
│ packages/transducers/src/xform/pluck.ts        │        2 │
│ packages/paths/test/index.ts                   │        2 │
│ packages/checks/src/exists.ts                  │        2 │
│ packages/transducers/src/iter/norm-range.ts    │        2 │
│ packages/transducers/src/xform/toggle.ts       │        2 │
│ packages/hiccup-svg/src/index.ts               │        2 │
│ packages/hiccup-svg/src/convert.ts             │        2 │
│ packages/checks/src/is-positive.ts             │        2 │
│ packages/transducers/src/xform/pad-last.ts     │        2 │
│ packages/sax/src/index.ts                      │        2 │
│ packages/transducers/src/xform/map.ts          │        2 │
│ packages/api/src/api/keyval.ts                 │        2 │
│ examples/async-effect/src/index.ts             │        2 │
│ examples/rotating-voronoi/src/stream-state.ts  │        2 │
│ packages/malloc/src/api.ts                     │        2 │
└────────────────────────────────────────────────┴──────────┘
```

check changed files for some specific authors

```sql
SELECT path
FROM file_changes
WHERE file_extension IN ('ts') AND author IN ('Karsten Schmidt')
GROUP BY path
ORDER BY path DESC
LIMIT 20;
```

```sh
┌─path────────────────────────────┐
│ tools/src/update-thing-links.ts │
│ tools/src/toc.ts                │
│ tools/src/search.ts             │
│ tools/src/readme.ts             │
│ tools/src/readme-examples.ts    │
│ tools/src/query-index.ts        │
│ tools/src/prune-changelogs.ts   │
│ tools/src/partials/user.ts      │
│ tools/src/partials/table.ts     │
│ tools/src/partials/package.ts   │
│ tools/src/partials/list.ts      │
│ tools/src/partials/link.ts      │
│ tools/src/partials/license.ts   │
│ tools/src/partials/examples.ts  │
│ tools/src/partials/docs.ts      │
│ tools/src/partials/blog.ts      │
│ tools/src/partials/asset.ts     │
│ tools/src/npm-stats.ts          │
│ tools/src/normalize-package.ts  │
│ tools/src/module-stats.ts       │
└─────────────────────────────────┘
```

## ref

- [December 2022 ClickHouse Bay Area Meetup - Clickhouse Git Import](https://youtu.be/G9MxRpKlbnI?t=5746)
