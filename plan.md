# MDEV-38735 — Plan: lossless binary round-trip for mysqldump --tab/--dir + mariadb-import

This document captures the motivation and the implementation plan agreed after the
preliminary review of https://github.com/MariaDB/server/pull/4610 (branch `hexblob`).
It is meant as the starting point for the next agent continuing this work.

## Background: the PR and the review

PR #4610 (MDEV-38735) proposed to extend `--hex-blob` to work in `mysqldump
--tab/--dir` mode (wrapping BLOB/BINARY columns with `HEX()`) and to add a new
`--hex-blob` option to `mariadb-import` (decoding via `UNHEX()` in the `LOAD DATA`
statement). The reviewer (gkodinov) requested changes with two main points:

1. Preferred direction: fix the *underlying* issue ("--tab not working with blobs in
   dump and import"), i.e. make the raw `SELECT INTO OUTFILE` → `LOAD DATA` pipeline
   lossless for binary data. "No new options. No new modes. Just fixing the base issue."
2. If a hex approach is kept at all, do it "right" with syntax options in `LOAD DATA`
   and `SELECT INTO OUTFILE` server-side — the client-side pre-parsing (regex-parsing
   `CREATE TABLE` in import_util.cc, extra per-table I_S queries, `UNHEX`/`SET` clause
   rewriting) breaks layering.

Formal review housekeeping requests (all still apply):

- Single commit in the PR (squash the current two).
- Commit message per CODING_STANDARDS.md; commit message, PR text and Jira must be
  in sync and describe exactly what is implemented.
- No whitespace-only changes.
- No conflicts with the base branch.
- All buildbot hosts must compile and run tests to completion.
- Test `--hex-blob` behavior in all relevant modes (normal INSERT mode, `--tab`,
  `--dir`, `--xml`), including the old-dump/new-import compatibility situation.

We decided (user decision, 2026-10-23): **fix the root cause**, revert the client-side
hex machinery.

## Motivation: what is actually broken

Code investigation of the current tree shows the premise of the original PR ("tab
separated text format cannot properly represent binary data, data gets corrupted or
truncated") is **incorrect for the default configuration**. The server-side pipeline
is byte-oriented and NUL-safe:

- Writer: `select_export::send_data` (sql/sql_class.cc:3661) escapes `0x00` as `\0`
  (`NEED_ESCAPING` macro at sql/sql_class.cc:3655 ends with `!(x)`; the escape is
  written at sql/sql_class.cc:3849). Escaping is pointer/length-based, no `strlen`
  anywhere, so embedded NULs cannot truncate output.
- Reader: `READ_INFO`/`read_sep_field` (sql/sql_load.cc) is fully byte-oriented,
  length-tracked; `unescape()` maps `\0` back to `0x00`. Binary-charset targets are
  forced to `my_charset_bin` (sql/sql_load.cc:1191, MDEV-11079), stores are
  length-based (`Field_blob::store` → `my_copy_8bit` memmove).
- Existing proof: MDEV-5481 test in mysql-test/main/mysqldump.test dumps GEOMETRY
  (WKB full of NULs) via `--hex-blob --tab` (hex-blob currently ignored in tab mode,
  i.e. raw binary) and reloads losslessly with plain `LOAD DATA INFILE`.

The **real** failure cases are configuration-dependent and stem from mysqldump
`--tab` emitting `INTO OUTFILE` *without* explicit `FIELDS`/`LINES` clauses and
mariadb-import emitting a *bare* `LOAD DATA INFILE`, so both sides silently depend on
session state:

1. `sql_mode=NO_BACKSLASH_ESCAPES`:
   - Writer disables escaping when no explicit `ESCAPED BY` was given
     (sql/sql_class.cc:3623-3625: `if (exchange->escaped->length() &&
     (exchange->escaped_given() || !(sql_mode & MODE_NO_BACKSLASH_ESCAPES)))`).
   - Reader symmetrically (sql/sql_load.cc:628-630).
   - Result: raw `\t`/`\n`/`\\` bytes inside BLOBs break the file structure on both
     sides. mysqldump --tab never emits `ESCAPED BY` (client/mysqldump.cc:4379-4386
     only appends clauses when the user passed `--fields-*` options).
2. Non-utf8mb4 `--default-character-set`: mysqldump emits `CHARACTER SET
   <default_charset>` into `INTO OUTFILE` (client/mysqldump.cc:4373-4376; only
   utf8mb4 is special-cased to `binary`). The server then converts binary columns to
   that charset and replaces unrepresentable bytes with `?` (strings/ctype-mb.c
   `my_copy_fix_mb`) — BLOB bytes >= 0x80 are destroyed.
3. Mismatched `--fields-*` options between dump and import sides (user error, but the
   default pinning below removes the common cases).

### Why the reviewer's "binary literal" idea does not work

gkodinov asked to explain why dumping binary literals (`0xABCDEF`) instead of hex
strings would not work (so import needs no changes): `LOAD DATA` performs no
expression evaluation — `0xDEADBEEF` in the data file is stored verbatim as the
9-character string into the target column, producing truncation/garbage. This was
demonstrated by the PR author in the PR discussion. (It *would* require extending
`LOAD DATA` syntax, which is the much bigger server-side change the reviewer's
fallback proposal envisions — not pursued here.)

## Plan

### Phase 0 — Branch preparation

1. Rebase `hexblob` onto current `origin/main` (reviewer: no conflicts with base).
2. Drop the two feature commits `13214cf5ea5` (mysqldump HEX() wrapping) and
   `ecb5b707f9f` (mariadb-import --hex-blob). Salvage for the new tests:
   mysql-test/main/mysqldump-hexblob.test and the additions to
   mysql-test/main/mariadb-import.test.
3. Final result is a single squashed commit, message per CODING_STANDARDS.md
   (MDEV-38735 prefix, problem/solution). Commit message, PR text and Jira must be
   kept in sync.

### Phase 1 — Prove the failure cases (before fixing)

Build the server, then write a scratch mtr test demonstrating with the current
(unfixed) code:

1. `sql_mode=NO_BACKSLASH_ESCAPES` + `mysqldump --tab` of a BLOB containing
   `\t`/`\n`/`0x00` → corrupted `.txt`.
2. `mysqldump --tab --default-character-set=utf8` → BLOB bytes >= 0x80 become `?`.
3. Default round-trip works (control) — confirms the pipeline is only
   configuration-fragile, not fundamentally broken.

### Phase 2 — mysqldump fix (client/mysqldump.cc)

In the tab/--dir mode query building (~lines 4367-4386):

1. **Pin the file format**: when the user gave no `--fields-*` /
   `--lines-terminated-by` options, always emit explicit
   `FIELDS TERMINATED BY '\t' ENCLOSED BY '' ESCAPED BY '\\' LINES TERMINATED BY '\n'`.
   Because `sql_exchange::escaped_given()` short-circuits the `NO_BACKSLASH_ESCAPES`
   check (sql/sql_class.cc:3623), this makes the writer sql_mode-independent.
   User-provided `--fields-*` options keep precedence (current behavior).
2. **Pin the write charset**: always emit `CHARACTER SET binary` (drop the utf8mb4
   special-case at mysqldump.cc:4373-4376) in tab mode, so no lossy
   binary→charset conversion can happen — each column is written verbatim in its own
   charset. (Decision made: go with the fully lossless variant; the alternative of
   keeping the current charset selection was considered and rejected because it
   leaves failure case 2 unfixed. Note: this changes `.txt` content for non-utf8mb4
   dumps — each column verbatim instead of converted. Must be called out in the
   commit message/PR/Jira.)
3. Verify interaction with `--headers` (the `UNION ALL` header trick, mysqldump.cc:4388-4398)
   and with user `--fields-*` overrides.

### Phase 3 — mariadb-import fix (client/mysqlimport.cc)

In `table_load_params::load_data()` (~lines 827-848):

1. Mirror the same pinned clauses when no `--fields-*` options are given:
   `FIELDS TERMINATED BY '\t' ENCLOSED BY '' ESCAPED BY '\\' LINES TERMINATED BY '\n'`
   plus `CHARACTER SET binary`, making the reader sql_mode-independent too.
2. Reassess the `SET collation_database=binary` workaround (mysqlimport.cc:817,
   MDEV-34890) — may become unnecessary or stay; test both ways.
3. Old-dump/new-import compatibility: `.txt` produced by an old mysqldump in the
   default configuration is byte-identical to the new output, so old backups still
   import (explicit review requirement). Add a test asserting this.

### Phase 4 — Revert & rework tests

1. Revert all changes to client/import_util.cc / import_util.h (regex CREATE TABLE
   parser) and the `--hex-blob` option in mysqlimport.cc; revert the HEX()-wrapping
   in mysqldump.cc. When reverting commit `13214cf5ea5`, also fix the pre-existing
   `insert_field_names` indentation regression it introduced (mysqldump.cc:3549:
   `dynstr_append_checked(&insert_field_names, last_name);` must stay unconditional,
   it feeds `insert_pat` used by `--complete-insert`).
2. New tests (extend mysql-test/main/mysqldump.test and mariadb-import.test, or a
   new dedicated test file):
   - Round-trip of all binary types (TINYBLOB..LONGBLOB, BINARY(n), VARBINARY(n),
     JSON) with `0x00`, `\t`, `\n`, `\`, and bytes 0x80-0xFF via `--tab` and `--dir`,
     verified by comparing table contents/checksums after export→import.
   - Round-trip under `NO_BACKSLASH_ESCAPES` (dump-side, import-side, and both).
   - Round-trip with `--default-character-set=utf8` and `latin1`.
   - User `--fields-enclosed-by` / `--fields-terminated-by` overrides still honored.
   - `--hex-blob` still ignored in tab mode (documented); existing MDEV-5481 geometry
     test unchanged; `--xml` and normal INSERT modes untouched.
3. No whitespace-only diffs anywhere (explicit review demand).

### Phase 5 — Validation & submission

1. Run: `./mtr main.mysqldump main.mariadb-import main.outfile_loaddata main.loaddata`
   plus a broad `main` client run.
2. All buildbot hosts must compile and run tests to completion (reviewer noted some
   hosts currently do not compile — removing the import_util.cc regex code should
   fix the failing hosts, verify).
3. Update the PR description and Jira MDEV-38735 to match the new approach (reviewer
   requires commit/PR/Jira sync), squash to a single commit, convert the PR back
   from draft.

## Key references

- PR: https://github.com/MariaDB/server/pull/4610 (draft, "need feedback")
- Reviewer: gkodinov; last review 2026-02-26; converted to draft 2026-04-23 after
  no reply for 3 weeks.
- Writer escaping: sql/sql_class.cc:3591-3625 (escape_char selection),
  sql/sql_class.cc:3655-3659 (NEED_ESCAPING), sql/sql_class.cc:3849 (`\0` escape),
  sql/sql_class.cc:3570 (write charset).
- Reader: sql/sql_load.cc:628-630 (escaped/NO_BACKSLASH_ESCAPES handling),
  sql/sql_load.cc:1191-1197 (binary charset forcing), sql/sql_load.cc:1425-1440
  (unescape).
- mysqldump tab query: client/mysqldump.cc:4343-4409.
- mariadb-import LOAD DATA: client/mysqlimport.cc:817-848.
- Existing passing binary round-trip test: mysql-test/main/mysqldump.test (~line
  2447, MDEV-5481) and mysql-test/main/outfile_loaddata.test (Bug#53088).
- CODING_STANDARDS.md in repo root.
