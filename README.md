# CodeIgniter 3 — PHP 8.5 emergency base

An **unofficial emergency fork** of the CodeIgniter 3 base framework, patched to
run on **PHP 8.5** (and 8.0–8.4).

The upstream CodeIgniter 3 branch is in maintenance mode and still targets PHP
5.6+. This repository keeps the CodeIgniter 3 API and application layout intact
while removing the framework's PHP 8.x fatal errors and deprecations, so legacy
CI3 applications can be moved onto a supported PHP runtime without a rewrite.

> Not affiliated with or endorsed by the CodeIgniter Foundation.
> The original upstream readme is preserved in [`readme.rst`](readme.rst).

## What is in this repository

This is the **base framework**, not an application:

* `system/` — the patched CodeIgniter 3 framework
* `application/` — the stock welcome application, useful only as a smoke test
* `tests/` — the CodeIgniter 3 PHPUnit suite
* [`PHP85-MIGRATION.md`](PHP85-MIGRATION.md) — every change made for PHP 8.5,
  plus instructions for applying this base to an existing application

There is no application-specific code here.

## Requirements

* PHP **8.5** recommended; 8.0–8.4 also supported
* Extensions: `mysqli`, `mbstring`, `openssl`, `session`, `tokenizer`, `xml`,
  `fileinfo`; `gd` only for `Image_lib`
* PHPUnit 9.x for the test suite (development only)

## Quick start

1. Copy `system/` and `index.php` into your CodeIgniter 3 project, or start from
   this repository and put your application in `application/`.
2. Configure your database. `mysqli` is recommended. The legacy `mysql` driver
   name still works: it is now a shim backed by mysqli. `mssql` is not
   available on PHP 8; use `sqlsrv` or `pdo/sqlsrv`.
3. Read [`PHP85-MIGRATION.md`](PHP85-MIGRATION.md) before upgrading an existing
   application — it lists the application-level changes you may still need to
   make.

## Status

* Framework syntax-checked on PHP 8.5.11.
* PHPUnit: 0 failures (database integration tests require live database servers
  and were skipped).
* Runtime smoke tests cover routing, loaders, helpers, sessions, encryption,
  XML-RPC, uploads, pagination, calendars, and validation without emitting
  notices or deprecations.

See the *Verification* and *Known limitations* sections of
[`PHP85-MIGRATION.md`](PHP85-MIGRATION.md) for details.

## Upstream

* CodeIgniter 3: <https://github.com/bcit-ci/CodeIgniter>
* CodeIgniter 4: <https://github.com/codeigniter4/CodeIgniter4>
* Documentation: <https://codeigniter.com/userguide3/>

## License

MIT — see [`license.txt`](license.txt). Upstream copyright notices are retained
in every file.
