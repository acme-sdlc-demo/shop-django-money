# Codebase map: django-money

## Purpose
SHOP's Django integration for money: `MoneyField` (a pair of Django model fields that stores an amount and its currency together), form fields, admin widgets, and an exchange-rate contrib app that fetches rates from Open Exchange Rates, Fixer or UniRate and stores them in the database. It extends py-moneyed's `Money` type with Django formatting, queryset support and validation. Used by SHOP's checkout, refunds and back-office price-editor services. [cite: source:product-note@r1] [cite: pyproject.toml]

---

## Solution layout

| Path | Purpose | Key dependencies |
|---|---|---|
| `djmoney/__init__.py` | Package version (`3.6.1`) | — [cite: djmoney/__init__.py] |
| `djmoney/money.py` | `Money` class — extends `moneyed.Money` with Django l10n, `F`-expression arithmetic, `decimal_places` propagation | `moneyed`, `django.utils.translation` [cite: djmoney/money.py] |
| `djmoney/settings.py` | Reads all `djmoney` Django-settings with defaults (`DEFAULT_CURRENCY`, `CURRENCY_CHOICES`, `DECIMAL_PLACES`, exchange keys, …) | `django.conf.settings`, `moneyed` [cite: djmoney/settings.py] |
| `djmoney/utils.py` | `MONEY_CLASSES`, `get_currency_field_name`, `get_amount`, `prepare_expression` | `djmoney.money`, `moneyed` [cite: djmoney/utils.py] |
| `djmoney/_compat.py` | `setup_managers` — wires `money_manager` on class-prepared signal | `djmoney.models.managers` [cite: djmoney/_compat.py] |
| `djmoney/models/fields.py` | `MoneyField` (DecimalField subclass), `CurrencyField` (CharField subclass), `MoneyFieldProxy` (descriptor), `MoneyValidator` | `djmoney.forms`, `djmoney.money`, `djmoney.settings` [cite: djmoney/models/fields.py] |
| `djmoney/models/managers.py` | `money_manager`, `understands_money`, `add_money_comprehension_to_queryset` — translates `Money` filter kwargs to `(amount, currency)` pairs | `djmoney.models.fields`, `djmoney.utils` [cite: djmoney/models/managers.py] |
| `djmoney/models/validators.py` | `MinMoneyValidator`, `MaxMoneyValidator` | `djmoney.money` [cite: djmoney/models/validators.py] |
| `djmoney/forms/fields.py` | `MoneyField` form field (MultiValueField) | `djmoney.money`, `djmoney.forms.widgets` [cite: djmoney/forms/fields.py] |
| `djmoney/forms/widgets.py` | `MoneyWidget` (MultiWidget), `HiddenMoneyWidget` | `django.forms` [cite: djmoney/forms/widgets.py] |
| `djmoney/contrib/exchange/models.py` | `ExchangeBackend`, `Rate` ORM models; `get_rate`, `_get_rate`, `convert_money` | `djmoney.settings` [cite: djmoney/contrib/exchange/models.py] |
| `djmoney/contrib/exchange/exceptions.py` | `MissingRate` exception | — [cite: djmoney/contrib/exchange/models.py] |
| `djmoney/contrib/exchange/backends/base.py` | `BaseExchangeBackend`, `SimpleExchangeBackend` | `certifi`, `django.db.transaction` [cite: djmoney/contrib/exchange/backends/base.py] |
| `djmoney/contrib/exchange/backends/openexchangerates.py` | `OpenExchangeRatesBackend` — fetches rates from openexchangerates.org | `djmoney.settings` [cite: djmoney/contrib/exchange/backends/openexchangerates.py] |
| `djmoney/contrib/exchange/backends/fixer.py` | `FixerBackend` — fetches rates from fixer.io | `djmoney.settings` [cite: djmoney/contrib/exchange/backends/fixer.py] |
| `djmoney/contrib/exchange/backends/unirateapi.py` | `UniRateBackend` — fetches rates from unirateapi.com | `djmoney.settings` [cite: djmoney/contrib/exchange/backends/unirateapi.py] |
| `djmoney/contrib/exchange/management/commands/update_rates.py` | `update_rates` management command (scheduled in production) | exchange backends [cite: djmoney/contrib/exchange/management/commands/update_rates.py] |
| `tests/settings.py` | Django settings for tests (SQLite, `djmoney.contrib.exchange`, `tests.testapp`) | — [cite: tests/settings.py] |
| `tests/settings_reversion.py` | Gate settings — appends `reversion` to INSTALLED_APPS | `tests.settings` [cite: tests/settings_reversion.py] |
| `tests/testapp/models.py` | Fixture models for all test scenarios | `djmoney.models.fields`, `djmoney.models.managers`, `djmoney.models.validators` [cite: tests/testapp/models.py] |
| `tests/test_money.py` | Unit tests for `Money` arithmetic, l10n, `decimal_places` propagation | — [cite: tests/test_money.py] |

---

## Key concepts

| Domain term | Type / file |
|---|---|
| **Money** | `djmoney.money.Money` — amount + currency, never separated [cite: djmoney/money.py] |
| **Currency** | ISO 4217 string; `moneyed.Currency`; validated in `CurrencyField` [cite: djmoney/models/fields.py] |
| **MoneyField** | `djmoney.models.fields.MoneyField` — a `DecimalField` that auto-creates a `CurrencyField` sibling named `<field>_currency` (or `currency_field_name`) [cite: djmoney/models/fields.py] |
| **MoneyFieldProxy** | Descriptor on the model that reconstructs `Money(amount, currency)` on attribute access [cite: djmoney/models/fields.py] |
| **money_manager** | Patches `Manager.get_queryset` so queryset methods accept `Money` filter values [cite: djmoney/models/managers.py] |
| **Exchange rate** | `Rate` row in `djmoney_exchange_rate` table; linked to an `ExchangeBackend`; fetched by `update_rates` [cite: djmoney/contrib/exchange/models.py] |
| **convert_money** | `djmoney.contrib.exchange.models.convert_money` — multiplies amount by DB-stored rate; requires `djmoney.contrib.exchange` in `INSTALLED_APPS` [cite: djmoney/contrib/exchange/models.py] |
| **Settlement rounding** | `Money.round(ndigits)` / built-in `round(money, n)` — delegates to `moneyed`; use half-even [cite: djmoney/money.py] |
| **Form MoneyField** | `djmoney.forms.fields.MoneyField` (MultiValueField) with `MoneyWidget` (amount + currency select) [cite: djmoney/forms/fields.py] |

---

## Where to change what

| Kind of change | Files to edit | Tests to add / check |
|---|---|---|
| Add/fix `Money` arithmetic or formatting | `djmoney/money.py` [cite: djmoney/money.py] | `tests/test_money.py` [cite: tests/test_money.py] |
| Add/fix model field behaviour (default, deconstruct, migration, null) | `djmoney/models/fields.py` [cite: djmoney/models/fields.py] | test file for the model area; run full gate |
| Add/fix queryset filtering with `Money` values | `djmoney/models/managers.py` [cite: djmoney/models/managers.py] | `tests/test_managers.py` |
| Add/fix money validators | `djmoney/models/validators.py` [cite: djmoney/models/validators.py] | tests covering `ValidatedMoneyModel` in `tests/testapp/models.py` [cite: tests/testapp/models.py] |
| Add/fix form field or widget | `djmoney/forms/fields.py`, `djmoney/forms/widgets.py` [cite: djmoney/forms/fields.py] [cite: djmoney/forms/widgets.py] | form/widget test file |
| Add/fix exchange-rate fetching or storage | `djmoney/contrib/exchange/backends/`, `djmoney/contrib/exchange/models.py` [cite: djmoney/contrib/exchange/models.py] | exchange contrib tests |
| Add a new exchange backend | Create `djmoney/contrib/exchange/backends/<name>.py`; export from `backends/__init__.py` [cite: djmoney/contrib/exchange/backends/base.py] | new backend test |
| Change global settings defaults | `djmoney/settings.py` [cite: djmoney/settings.py] | tests that rely on those defaults |
| Add fixture model for a new test scenario | `tests/testapp/models.py` [cite: tests/testapp/models.py] | matching test function `test_*` |

---

## Build and test

```
# Install (required once per worktree):
$python -m pip install -e .

# Gate (run before every commit, enforced on CI):
$python -m black --check djmoney tests
$python -m isort --check-only --skip-glob "*.pyi" djmoney tests
$python -m flake8 djmoney tests
$python -m pytest --ds=tests.settings_reversion

# Run only tests for a file area (still use the gate settings):
$python -m pytest --ds=tests.settings_reversion tests/test_money.py
```

[cite: pyproject.toml] [cite: tests/settings_reversion.py]

---

## Risky areas

- **`MoneyField.contribute_to_class`** — auto-creates the `CurrencyField` sibling and sets `MoneyFieldProxy`; the `creation_counter` increment is load-order sensitive. Changes here risk breaking migrations and proxy-model handling. [cite: djmoney/models/fields.py]
- **`money_manager` / `understands_money`** — patches manager and queryset instances in place at class-prepare time. Any change that alters how `Q` objects or `F` expressions are expanded can silently break filtering across every model. [cite: djmoney/models/managers.py]
- **`Money._copy_attributes`** — propagates `decimal_places` through arithmetic; missing it produces silent precision loss. [cite: djmoney/money.py]
- **`MoneyField.deconstruct`** — must emit no spurious kwargs; any regression produces unnecessary migrations on every deploy. [cite: djmoney/models/fields.py]
- **Exchange rate caching** — production uses no cache layer; Unverified: whether a future cache layer would invalidate correctly on `update_rates`. [cite: source:transcript-run-0ffa3540@r1]
- **`CurrencyField` null parity** — `MoneyFieldProxy.__set__` raises `ValueError` if amount is non-null but currency is null; the `null` flags on both fields must stay in sync. [cite: djmoney/models/fields.py]

---

## Connections

### Provides
- `djmoney.money` (component): django-money's `Money` subclass of `moneyed.Money`, with Django-aware formatting and `F`-expression support; paths: `djmoney/money.py` [cite: djmoney/money.py]
- `djmoney.models.moneyfield` (component): `MoneyField` model field — stores price, total and refund amounts in SHOP service databases; paths: `djmoney/models/fields.py` [cite: djmoney/models/fields.py]
- `djmoney.forms.moneyfield` (component): `MoneyField` form field + `MoneyWidget` — validates and renders money inputs in checkout and admin; paths: `djmoney/forms/fields.py`, `djmoney/forms/widgets.py` [cite: djmoney/forms/fields.py]
- `djmoney.contrib.exchange` (component): exchange-rate storage and `convert_money`; paths: `djmoney/contrib/exchange/` [cite: djmoney/contrib/exchange/models.py]

### Uses
- `money.type` from shop-py-moneyed: `moneyed.Money`, `moneyed.Currency`, ISO 4217 currency table, arithmetic and rounding — base money type this package extends [cite: djmoney/money.py]
- `money.formatting` from shop-py-moneyed: `moneyed.l10n.format_money` — locale-aware display formatting [cite: djmoney/money.py]
- ? from openexchangerates.org: HTTP API for fetching live exchange rates (called by `OpenExchangeRatesBackend`) [cite: djmoney/contrib/exchange/backends/openexchangerates.py]
- ? from fixer.io: HTTP API for fetching live exchange rates (called by `FixerBackend`) [cite: djmoney/contrib/exchange/backends/fixer.py]
- ? from unirateapi.com: HTTP API for fetching live exchange rates (called by `UniRateBackend`) [cite: djmoney/contrib/exchange/backends/unirateapi.py]
