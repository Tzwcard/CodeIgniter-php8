# CodeIgniter 3 — PHP 8.5 migration notes

This document describes the changes made to the base CodeIgniter 3 framework in
this repository so that it runs cleanly on **PHP 8.5**, and explains how to apply
the same base to an existing CodeIgniter 3 application.

This is a framework-level port. No application code is included; the bundled
`application/` directory is still the stock CodeIgniter 3 welcome app.

## Target and verified environment

| Item | Value |
| --- | --- |
| Framework | CodeIgniter 3.2.0-dev (`develop`) |
| Target PHP | 8.5+ (tested on **PHP 8.5.11 NTS**, Windows x64) |
| Supported PHP | 8.0 – 8.5 (the backward-compatibility guards also keep the code valid on 7.x, but PHP < 8.0 is no longer supported) |
| PHPUnit | 9.6.38 |
| Extensions used in verification | mysqli, mbstring, openssl, session, tokenizer, xml, fileinfo |

Every guard added below is written as `PHP_VERSION_ID < 8xxxx`, so the same
source tree still works on PHP 8.0–8.4 and the behaviour on older versions is
unchanged.

## Compatibility changes

### 1. `E_STRICT` is deprecated (PHP 8.4)

Referencing the `E_STRICT` constant now emits `E_DEPRECATED`.

* `index.php` — removed `~E_STRICT` from the production/testing
  `error_reporting()` mask.
* `system/core/Exceptions.php` — the error-level label map now uses the literal
  value `2048` instead of the constant.
* `tests/Bootstrap.php` — `error_reporting(E_ALL)` instead of
  `E_ALL | E_STRICT`.

### 2. Dynamic properties (PHP 8.2+)

CodeIgniter intentionally decorates framework objects at runtime (controllers and
models receive loaded libraries, the loader mirrors controller properties, and
several classes assign properties in their constructors). PHP 8.2 deprecated
creating undeclared dynamic properties, so:

* `system/core/Controller.php`, `system/core/Model.php`,
  `system/core/Loader.php` carry `#[\AllowDynamicProperties]`.
  The attribute is inherited, so application controllers/models extending
  `CI_Controller` / `CI_Model` inherit it too.
* Properties that were assigned but never declared are now declared where that
  is safe and clearer:
  * `CI_Router::$uri` (`system/core/Router.php`)
  * `CI_URI::$config` (`system/core/URI.php`)
  * `CI_Image_lib::$dest_image` (`system/libraries/Image_lib.php`)
  * `CI_Table::$temp` (`system/libraries/Table.php`)

### 3. Implicitly nullable parameter types (PHP 8.4)

`array $params = NULL` is deprecated; the type must be explicitly nullable.

* `system/libraries/Encryption.php` — `CI_Encryption::encrypt()` and
  `decrypt()` now use `?array $params = NULL`.

### 4. No-op functions deprecated in PHP 8.5

These functions have been no-ops since PHP 8.0/8.1 and are deprecated in 8.5.
The framework only calls them on older PHP versions:

* `system/libraries/Image_lib.php` — all `imagedestroy()` calls go through the
  new `_destroy_gd()` helper (`PHP_VERSION_ID < 80500`).
* `system/helpers/captcha_helper.php` — the CAPTCHA image handle is only freed
  with `ImageDestroy()` on PHP < 8.5.
* `system/libraries/Upload.php` — `finfo_close()` is guarded (`< 80500`).
* `system/libraries/Xmlrpc.php` — `xml_parser_free()` calls go through
  `_free_xml_parser()` (`< 80500`); `system/libraries/Xmlrpcs.php` uses the same
  helper.

### 5. XML parser is an object in PHP 8

Since PHP 8, `xml_parser_create()` returns an `XMLParser` object instead of a
resource. The XML-RPC code used the parser as an array key via `(string) $parser`
(which now throws "Object ... could not be converted to string") and used
`xml_set_object()` / string handler names, both deprecated.

* `system/libraries/Xmlrpc.php`
  * New `_parser_key()` helper returns `spl_object_id($parser)` for objects and
    the string form for old resources; it is used everywhere the parser was
    previously cast to string (`parseResponse()`, `open_tag()`, `closing_tag()`,
    `character_data()`).
  * Removed `xml_set_object()`; handlers are registered as callable arrays
    (`array($this, 'open_tag')`, etc.).
* `system/libraries/Xmlrpcs.php`
  * Same `_parser_key()` change and callable-array handlers.

### 6. `session.sid_length` is deprecated (PHP 8.4)

`system/libraries/Session/Session.php` only calls
`ini_set('session.sid_length', ...)` on PHP < 8.4. On 8.4+ the framework leaves
the generated session ID length to PHP's configuration. If an application
depends on a minimum of 160 bits of SID entropy while using
`session.sid_bits_per_character = 4`, review the php.ini settings (PHP 8.4 also
deprecated both SID INI settings) or use the default configuration.

### 7. MySQL driver now uses mysqli

The `mysql_*` extension was removed in PHP 7. The `mysql` driver is kept as a
**compatibility shim** so existing configurations continue to work:

* `system/database/drivers/mysql/mysql_driver.php` extends
  `CI_DB_mysqli_driver`.
* `system/database/drivers/mysql/mysql_result.php` extends
  `CI_DB_mysqli_result`.
* `system/database/drivers/mysql/mysql_utility.php` extends
  `CI_DB_mysqli_utility`.
* `mysql_forge.php` is unchanged (it shares the mysqli forge API).

A configuration with `'dbdriver' => 'mysql'` therefore loads
`CI_DB_mysql_driver`, still reports `dbdriver` as `mysql`, and performs all work
through **mysqli**. The mysqli extension (and `mysqlnd` for native prepared
statements) must be installed. New configurations should use `mysqli` directly.

### 8. mssql driver

The `mssql` extension was removed in PHP 7 and cannot be emulated by the
`sqlsrv` driver. `system/database/DB.php` now fails with an explanatory
`show_error()` message instead of an "undefined function" fatal. Use `sqlsrv`
or `pdo/sqlsrv` instead.

### 9. `highlight_code()` output change (PHP 8.3+)

Since PHP 8.3, `highlight_string()` wraps output in
`<pre><code style="...">` without newlines and emits an artificial trailing
`?>` span. `system/helpers/text_helper.php` now strips both the old and new
wrappers, so `highlight_code()` keeps returning the historical
`<code>...</code>` shape. The corresponding unit test is version-aware.

### 10. Tooling metadata

* `composer.json` — `"php": ">=8.0"` (was `>=5.4.8`).
* `.github/workflows/test-phpunit.yml` — CI matrix is now PHP
  `8.5, 8.4, 8.3, 8.2, 8.1, 8.0`; the PHP 5.4/5.5/5.6 `mysql` entries were
  removed.
* `.gitattributes` — PHP files and tests are checked out with LF so
  line-ending-sensitive test fixtures (heredocs) behave the same on Windows as
  on CI.

## Applying this base to an existing CodeIgniter 3 application

1. Replace your `system/` directory with the one from this repository. Keep your
   `application/`, `assets/`, and entry-point customizations.
2. Keep (or merge) your top-level `index.php`. If you use a customized
   `index.php`, remove `E_STRICT` from the `production`/`testing`
   `error_reporting()` mask.
3. Ensure these extensions are available: `mysqli`, `mbstring`, `openssl`,
   `session`, `tokenizer`, `xml`, and `fileinfo` (for upload MIME detection).
   `gd` is only needed if you use `Image_lib`.
4. Database configuration:
   * `mysql` continues to work through the mysqli shim; `mysqli` is preferred
     for new configurations.
   * `mssql` is no longer usable; migrate to `sqlsrv` / `pdo/sqlsrv`.
   * `mcrypt` has been removed from PHP; `Encryption` automatically falls back to
     OpenSSL. Explicitly selecting `'driver' => 'mcrypt'` logs an error and uses
     OpenSSL.
5. Scan application code for the same PHP 8 hazards fixed here:
   * Remove direct calls to `mysql_*`, `mssql_*`, `mcrypt_*`.
   * Replace `(string) $xmlParser` with an object-safe key; remove
     `xml_set_object()` and string handler names.
   * Replace `imagedestroy()`, `finfo_close()`, `xml_parser_free()`,
     `curl_close()`, etc. — they are no-ops and deprecated in 8.5.
   * Change `array $x = NULL` to `?array $x = NULL`.
   * Declare properties or add `#[\AllowDynamicProperties]` to your own classes
     that rely on dynamic properties.
6. Run the test suite (or at least your application smoke tests) with
   `error_reporting(E_ALL)` and a deprecation-to-exception handler so remaining
   PHP 8.x notices surface early.

## Verification performed

* **Syntax check:** 192 PHP files (framework, bundled application, and
  `index.php`) linted on PHP 8.5.11 — 0 syntax errors.
* **Static scan:** a tokenizer-based scan for removed functions, removed
  constants, implicit nullable parameters, and non-canonical casts found no
  remaining framework issues.
* **PHPUnit:** 337 tests, 1094 assertions, **0 failures**, 11 warnings,
  24 skipped (database integration tests that need a live server). The warnings
  come from `assertObjectHasAttribute()` in the test harness, kept for
  compatibility with older PHPUnit versions.
* **Runtime smoke tests** through the PHP built-in server:
  * Welcome page and routed controller render without notices or deprecations.
  * A controller exercising `URI`/`Router`/`Loader`, helpers, `Table`,
    `Encryption` (OpenSSL round-trip), XML-RPC payload creation, `Session`,
    `Form_validation`, `Pagination`, `Calendar`, `User_agent`, `Input`, and
    `Output` returns clean JSON.
  * `Encryption` with an explicit `mcrypt` request falls back to OpenSSL.
  * `Upload::_file_mime_type()` runs through the guarded `finfo_close()` path.
  * XML-RPC parsing handles both a valid payload and a malformed one (returns a
    fault response) without deprecations.
  * The `mysql` driver loads as a subclass of `CI_DB_mysqli_driver` while
    reporting `dbdriver = mysql`.

## Known limitations

* No live MySQL/PostgreSQL/sqlsrv server was available during verification, so
  the database drivers were checked structurally and through the driver-loading
  harness, not with end-to-end queries.
* The verification PHP build has no GD extension, so `Image_lib` image
  operations were reviewed and linted but not executed; the `imagedestroy()`
  guard is the only 8.5-specific change there.
* `mssql` is intentionally unsupported.
* The `mysql` shim inherits mysqli behaviour; application code that reached into
  `mysql_*`-specific behaviour (for example `mysql_pconnect`) should be
  reviewed.
* PHP versions below 8.0 are no longer supported by `composer.json` or the CI
  matrix, even though most compatibility guards remain valid there.
