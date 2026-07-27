# Quality Gates & CI/CD

> The full pipeline: PHPCS + PHPCompatibilityWP, PHPStan, PHPUnit matrix, coverage/mutation gates, Plugin Check, QIT, and the GitHub Actions workflow that enforces all of it.

Individual tools are configured in their own guidelines (`09-testing.md`, `10-static-analysis.md`, `14-e2e-testing.md`). This file defines the **gates** — what must pass, in what order, before code merges.

---

## Gate Order (Cheap → Expensive)

```
1. composer cs                 # PHPCS + PHPCompatibilityWP     (~seconds)
2. composer analyse            # PHPStan level 8                (~seconds)
3. composer test               # Unit tests, PHP 8.0–8.4 matrix (~seconds)
4. composer coverage:check     # ≥ 80% on src/Services          (~seconds)
5. vendor/bin/infection        # MSI ≥ 70                       (~minutes)
6. wp plugin check             # Plugin Check (PCP)             (~seconds)
7. Integration + E2E           # wp-env + Playwright            (~minutes)
```

Locally, run 1–4 before every commit (see `11-checklist.md`). 5–7 run in CI on every PR; a failure at any gate blocks merge.

---

## PHPCS + PHPCompatibilityWP

`13-automation-tooling.md` references `phpcs.xml.dist` — this is its canonical content. `PHPCompatibilityWP` mechanically catches syntax/function usage above the declared PHP minimum (the "Features above 8.0 documented" checklist row).

```bash
composer require --dev wp-coding-standards/wpcs:^3.4 \
    phpcompatibility/phpcompatibility-wp:^2.1 \
    dealerdirect/phpcodesniffer-composer-installer:^1.0
```

> **Do not force `squizlabs/php_codesniffer:^4.0`.** WPCS 3.4 requires PHPCS `^3.13.5`; PHPCS 4.x is not supported yet. Let Composer resolve PHPCS transitively.

### phpcs.xml.dist

```xml
<?xml version="1.0"?>
<ruleset name="NVM Plugin">
	<description>Nevma WordPress/WooCommerce coding standards.</description>

	<file>src</file>
	<file>nvm-plugin.php</file>

	<arg name="extensions" value="php"/>
	<arg name="parallel" value="8"/>
	<arg value="sp"/>

	<!-- PHP compatibility: minimum 8.0, no upper bound. -->
	<config name="testVersion" value="8.0-"/>
	<rule ref="PHPCompatibilityWP"/>

	<!--
	Minimum WP version the deprecation sniffs check against. Set it explicitly —
	the WPCS default moves every release (6.7 as of WPCS 3.4) and would silently
	change your findings on upgrade. Keep it in sync with the plugin header's
	"Requires at least".
	-->
	<config name="minimum_wp_version" value="6.7"/>

	<rule ref="WordPress">
		<!-- PSR-4 class files (src/Services/Stock_Service.php), not class-*.php. -->
		<exclude name="WordPress.Files.FileName"/>
		<!-- Short array syntax is the project standard. -->
		<exclude name="Universal.Arrays.DisallowShortArraySyntax"/>
	</rule>

	<rule ref="WordPress.WP.I18n">
		<properties>
			<property name="text_domain" type="array">
				<element value="nvm-plugin"/>
			</property>
		</properties>
	</rule>

	<!--
	Prefixes must be at least 4 characters (WPCS 3.2+). A bare "NVM" is now
	rejected as too short — and a rejected prefix is dropped entirely, so every
	NVM-prefixed global would then be reported as unprefixed. Declare the
	plugin's full namespace instead.
	-->
	<rule ref="WordPress.NamingConventions.PrefixAllGlobals">
		<properties>
			<property name="prefixes" type="array">
				<element value="nvm_plugin"/>
				<element value="NVM\Plugin"/>
			</property>
		</properties>
	</rule>
</ruleset>
```

```json
{
	"scripts": {
		"cs": "phpcs",
		"cs:fix": "phpcbf"
	}
}
```

### Upgrading WPCS 3.1 → 3.4

Expect new findings on the first run after the bump. These are the changes that produce them — none are auto-fixable by `phpcbf`:

| Change | Since | Effect |
|--------|-------|--------|
| Minimum prefix length raised 3 → 4 chars | 3.2 | `NVM` is rejected (`ShortPrefixPassed`) **and discarded**, cascading into `PrefixAllGlobals` errors on every global. Use the full namespace — see the ruleset above. |
| `minimum_wp_version` default 6.2 → 6.7 | 3.2–3.4 | Many more deprecated-function/parameter warnings. Pin the value explicitly rather than inheriting the default. |
| New `WordPress.WP.GetMetaSingle` sniff | 3.2 | Warns on `get_*_meta()` / `get_metadata*()` called with a key but no `$single` argument. Pass `$single` explicitly — the return type differs. |
| `wp_kses_allowed_html()` dropped from escaping functions | 3.3 | Now reports `EscapeOutput.OutputNotEscaped`; it never escaped anything. |
| `@parse_url()` no longer accepted | 3.4 | `NoSilencedErrors` flags it. `parse_url()` returns `false` on failure — check the return value. |
| Deprecated-WP detection extended to WP 7.0 | 3.4 | New `DeprecatedFunctions` hits for anything WP removed recently. |
| `WordPress.PHP.POSIXFunctions` deprecated | 3.3 | Remove it from any ruleset that references the sniff by name. |
| `allow_single_item_single_line_associative_arrays` deprecated | 3.4 | Renamed to `allow_single_item_single_line_explicit_key_arrays` (behaviour identical). |

Triage by sniff name — the ruleset's `<arg value="sp"/>` already prints them in `composer cs` output. Fix findings rather than excluding sniffs; if an exclusion is genuinely warranted, comment *why* in `phpcs.xml.dist`.

---

## Plugin Check (PCP)

The official WordPress.org quality checker — catches escaping misses, enqueued-asset mistakes, i18n issues, and wp.org guideline violations:

```bash
# Locally via wp-env.
npx wp-env run cli wp plugin install plugin-check --activate
npx wp-env run cli wp plugin check nvm-plugin
```

Gate: **no ERROR-level findings**. Triage WARNINGs — fix or document why they're false positives.

---

## QIT (WooCommerce Quality Insights Toolkit)

WooCommerce's official managed test suite — runs your built ZIP against clean WordPress + WooCommerce environments in the cloud. Requires a WooCommerce.com (partner) account: `qit partner:add`.

```bash
composer require --dev woocommerce/qit-cli

# Core checks against the built ZIP.
./vendor/bin/qit run:activation nvm-plugin --zip=dist/nvm-plugin.zip
./vendor/bin/qit run:security nvm-plugin --zip=dist/nvm-plugin.zip
./vendor/bin/qit run:phpstan nvm-plugin --zip=dist/nvm-plugin.zip
./vendor/bin/qit run:phpcompatibility nvm-plugin --zip=dist/nvm-plugin.zip
./vendor/bin/qit run:api nvm-plugin --zip=dist/nvm-plugin.zip
```

Run before every release (it tests the **built artifact**, catching build.sh mistakes local tools can't) — and `run:activation` against WC release candidates when a major WooCommerce version is announced.

---

## GitHub Actions Workflow

`14-e2e-testing.md` covers the E2E job. This is the complete pipeline:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          coverage: none
      - run: composer install --prefer-dist --no-progress
      - name: PHPCS + PHPCompatibility
        run: composer cs
      - name: PHPStan
        run: composer analyse

  unit:
    needs: lint
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        php: ['8.0', '8.1', '8.2', '8.3', '8.4']
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php }}
          coverage: pcov
      - run: composer install --prefer-dist --no-progress
      - name: Unit tests
        run: composer test:unit
      - name: Coverage gate (single PHP version is enough)
        if: matrix.php == '8.3'
        run: |
          composer test:coverage
          composer coverage:check

  mutation:
    needs: unit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          coverage: pcov
      - run: composer install --prefer-dist --no-progress
      - name: Infection (MSI gate)
        run: vendor/bin/infection --threads=max --no-progress

  plugin-check:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Plugin Check
        uses: WordPress/plugin-check-action@v1
        with:
          build-dir: '.'

  e2e:
    needs: unit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - name: Start WordPress
        run: npm run env:start
      - name: Seed test data
        run: bash tests/e2e/bin/seed.sh
      - name: Integration tests
        run: |
          npx wp-env run tests-cli --env-cwd=wp-content/plugins/nvm-plugin \
            vendor/bin/phpunit -c phpunit-integration.xml
      - name: E2E tests
        run: npm run test:e2e
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: test-results/playwright-report/
```

Notes:

- **PCOV over Xdebug in CI** — same coverage data, several times faster.
- The PHP matrix is what actually enforces "PHP 8.0+ compatible" — PHPCompatibilityWP catches syntax, the matrix catches runtime behavior.
- Branch protection on `main` must require all five jobs.

---

## Release Gate

Before tagging a release, in addition to green CI:

```bash
bash build.sh                                                    # Build dist ZIP.
./vendor/bin/qit run:activation <slug> --zip=dist/<slug>.zip     # Artifact smoke test.
./vendor/bin/qit run:security  <slug> --zip=dist/<slug>.zip
```

The built ZIP — not the repo — is what ships. Test it.
