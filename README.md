# config-platform-override

## Feature exercised

`config.platform` in `composer.json` instructs Composer to treat the declared platform package versions as canonical during dependency resolution, regardless of the actual PHP runtime version or installed extensions. The faked values are recorded in the lockfile's `platform-overrides` block.

## config.platform semantics

When `config.platform` is set, Composer substitutes the declared fake versions for all platform packages (`php`, `ext-*`, `lib-*`) during the SAT solver pass. This means:

- A package that requires `php: >=8.2` will be **excluded** if `config.platform.php` is set to `8.1.0`, even if the real runtime is PHP 8.3.
- A package that requires `ext-mbstring: *` will be satisfied as long as `config.platform.ext-mbstring` is declared, even if the extension is not installed.

The resolver uses the faked values; the real runtime is not consulted.

### Lockfile representation

Composer records two distinct blocks in `composer.lock`:

| Block | Contents | Purpose |
|---|---|---|
| `platform` | `{"php": ">=8.1"}` | Constraints from the root `require` block (not the faked values) |
| `platform-overrides` | `{"php": "8.1.0", "ext-mbstring": "1.0.0"}` | The faked values from `config.platform` |

`config.platform` affects **which versions get resolved**, not what appears in the `platform` block. The `platform` block reflects the requirement constraints. The `platform-overrides` block is what Mend must read to understand what platform was faked.

## Resolved monolog/monolog version and why

`monolog/monolog 3.10.0` was resolved. monolog ^3.0 requires `php: >=8.1`. With `config.platform.php = 8.1.0`, the faked version satisfies this constraint, so the latest 3.x release at time of generation (3.10.0) is selected. monolog 3.x dropped PHP <8.1, and 3.10.0 is the highest stable 3.x release available. The `psr/log ^2.0 || ^3.0` requirement of monolog is satisfied by `psr/log 3.0.2`.

If the host PHP were older than 8.1 but `config.platform.php` were not set, Composer would refuse to install monolog 3.x. The `config.platform` override is what makes this resolution succeed regardless of the real runtime.

## Assertion contract

The faked platform MUST NOT leak into Mend's dependency tree. Specifically:

1. `php` must not appear as a dependency tree node.
2. `ext-mbstring` must not appear as a dependency tree node.
3. Only `monolog/monolog:3.10.0` (direct) and `psr/log:3.0.2` (transitive) appear in the tree.
4. Both packages must have `source = registry` (Packagist).
5. Mend must read the resolved versions from `composer.lock` — not re-resolve using the host PHP version.
6. Mend must honor the `platform-overrides` block in the lockfile when interpreting what platform was active during resolution.

## Expected dependency tree

### Direct dependencies (`require`)

| Package | Version | Source |
|---|---|---|
| `monolog/monolog` | 3.10.0 | registry (Packagist) |

### Transitive dependencies

| Package | Version | Source | Parent |
|---|---|---|---|
| `psr/log` | 3.0.2 | registry (Packagist) | monolog/monolog |

### Platform packages (must NOT appear in tree)

| Package | Faked version | Reason excluded |
|---|---|---|
| `php` | 8.1.0 | Platform package — not an installable Packagist library |
| `ext-mbstring` | 1.0.0 | Platform package — not an installable Packagist library |

### Summary

- Total installable packages in lockfile: **2**
- Direct (`require`): **1**
- Transitive: **1**
- Dev packages (`require-dev`): **0**

## Mend failure modes exercised

- **Mend uses host PHP instead of config.platform.php**: Mend re-resolves against the real runtime, potentially selecting a different version or failing resolution entirely.
- **Mend ignores platform-overrides**: Mend does not read `platform-overrides` from the lockfile, missing the distinction between faked and real platform.
- **Faked platform leaks into the tree**: `php` or `ext-mbstring` appears as a dependency node in Mend's output — violating the platform-exclusion contract.
- **Wrong version selected**: If Mend re-resolves, it may pick a version incompatible with the faked platform constraint.

## Probe metadata

```
pattern:                config-platform-override
pm:                     composer
lockfile:               composer.lock
config.platform.php:    8.1.0
config.platform.ext-mbstring: 1.0.0
resolved_monolog:       3.10.0
php_minimum:            8.1
composer_minimum:       2.6
generated:              2026-04-29
target:                 remote
repo:                   https://github.com/mend-detection-qa/composer-config-platform-override-probe
```