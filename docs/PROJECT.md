# Project: django-money (SHOP fork)

## Purpose and users

SHOP's Django integration for money: `MoneyField` on models, form fields, and admin widgets that store
and validate prices, order totals, and refund amounts in SHOP's Django services (checkout, refunds, and
back-office price editor). [cite: profile:purpose] [cite: source:product-note@r1]

The library also ships `djmoney.contrib.exchange`, an optional app that fetches exchange rates from
external providers and stores them in the service's own database so that money values can be converted
between currencies. Three backends are provided: `OpenExchangeRatesBackend` (primary in production),
`FixerBackend` (fallback), and `UniRateBackend` (present in code; whether SHOP uses it is unconfirmed
— see OPEN_QUESTIONS.md Q4). [cite: source:transcript-run-0ffa3540@r1]
[cite: djmoney/contrib/exchange/backends/openexchangerates.py]
[cite: djmoney/contrib/exchange/backends/fixer.py]
[cite: djmoney/contrib/exchange/backends/unirateapi.py]

Direct consumers are backend Python engineers building or maintaining SHOP's Django services. The
library is a fork of upstream django-money; SHOP takes it from a git tag rather than from PyPI.
[cite: source:how-we-release@r1]

## Domain and constraints

**Regulations and safety:** None — retail software, no regulated domain.
[cite: profile:regulatory_context] [cite: profile:safety_class]

**Platform:** Django 4.2–5.2, Python 3.10–3.13 (also PyPy). SHOP production services run
Django 5.2 on Python 3.12. [cite: pyproject.toml]
[cite: source:transcript-run-0ffa3540@r1]

**Core dependency:** `py-moneyed >= 2.0, < 3.1` (SHOP's `shop-py-moneyed` project) supplies `Money`
and `Currency` types, the ISO 4217 currency table, arithmetic, rounding and allocation. This library
extends `moneyed.Money` in `djmoney/money.py` with Django-aware formatting and `F`-expression support.
[cite: pyproject.toml]
[cite: djmoney/money.py]

**Exchange optional dependency:** only `certifi` (TLS trust for outbound rate-fetch requests);
all HTTP transport is done with the standard library. No other third-party library is introduced by
the exchange app. [cite: pyproject.toml]
[cite: djmoney/contrib/exchange/backends/base.py]

**Key invariants from the SHOP domain:**
- A `Money` value is an amount together with its currency; a bare `Decimal` meant as money is a defect.
  [cite: profile:languages]
- A blank or missing currency on a stored field is a defect, not an acceptable default.
  [cite: source:product-note@r1]
- Prices are stored in the currency they were set in; silent conversion on save is not allowed.
  [cite: source:product-note@r1]

**Performance:** The code in `get_rate()` calls Django's cache (`cache.get` / `cache.set`) with a
default timeout of 600 s (`RATES_CACHE_TIMEOUT`). Carol stated "No cache layer" in the interview,
which may mean SHOP services configure a no-op Django cache backend. This is unresolved — see
OPEN_QUESTIONS.md Q3 and Documents vs code Discrepancy 1.
[cite: djmoney/settings.py]
[cite: djmoney/contrib/exchange/models.py]
[cite: source:transcript-run-0ffa3540@r1]

## Decisions so far (and why)

**Two database columns per money value.** `MoneyField` always creates a paired `CurrencyField` on the
model, so amount and currency are stored together and never separated. This directly enforces the
"Money = amount + currency, never one without the other" invariant from the domain.
[cite: djmoney/models/fields.py]
[cite: source:product-note@r1]

**`DEFAULT_CURRENCY` defaults to `None`; every `MoneyField` must name its currency explicitly.**
The library's global default is `None`, so omitting `default_currency=` on a `MoneyField` that also
has a numeric default triggers an `AssertionError` at class-definition time. SHOP enforces the
explicit-currency convention through code review, not by setting a project-wide default.
[cite: djmoney/settings.py]
[cite: djmoney/models/fields.py]
[cite: source:transcript-run-0ffa3540@r1]

**`default_currency` and `currency_choices` are excluded from migrations (`non_db_attrs`).**
These two attributes affect application behaviour but do not change the database schema, so they are
omitted from `deconstruct()` and do not trigger spurious migration files when they change. This
satisfies the requirement for stable migrations.
[cite: djmoney/models/fields.py]
[cite: source:product-note@r1]

**Form field uses a `ChoiceField` for currency.** The form-layer `MoneyField` pairs a `DecimalField`
for the amount with a `ChoiceField` for the currency. Submitting a currency code not in the allowed
choices raises a `ValidationError` from the `ChoiceField`.
[cite: djmoney/forms/fields.py]
[cite: source:product-note@r1]

**Exchange rates stored in the database via ORM models.** The `ExchangeBackend` and `Rate` models
own the rate data. The `update_rates` management command runs on a schedule in each shop service to
refresh them. Open Exchange Rates is primary; Fixer is the fallback. API keys are injected by the
deployment environment and never committed to the repository.
[cite: source:transcript-run-0ffa3540@r1]
[cite: djmoney/contrib/exchange/models.py]

**Releases from the default branch by git tag.** SHOP does not publish to PyPI. A release is cut
after every change on the default branch has been merged through the platform; QA signs off and a
product owner approves before the platform tags the release. Upstream's PyPI and RELEASE.rst process
does not apply.
[cite: source:how-we-release@r1]

**Upcoming: allocation display in admin.** Once shop-py-moneyed gains allocation support, the admin
will be extended to display allocations (split bills, refunds across order lines).
[cite: source:product-note@r1]

## Glossary

Terms are used with the meanings in the SHOP domain glossary [cite: profile:languages]; the entries
below note how each term maps to a library concept.

| Term | Library concept |
|---|---|
| **Money** | `moneyed.Money` from shop-py-moneyed — an amount (`Decimal`) plus a `Currency`. `djmoney.money.Money` is django-money's subclass, adding Django-aware formatting and `F`-expression support. Never stored or passed without both parts. |
| **Currency** | An ISO 4217 code; stored as a `CurrencyField` (a `CharField`) alongside every `MoneyField`. |
| **Money field** | `djmoney.models.fields.MoneyField` — the pair of `DecimalField` (amount) + `CurrencyField` (currency) on a Django model. |
| **Exchange rate** | A `Rate` ORM record: a `Decimal` value linked to an `ExchangeBackend` that records the base currency and last-update time. |
| **Rate provider** | A `BaseExchangeBackend` subclass; `OpenExchangeRatesBackend` (primary in production), `FixerBackend` (fallback), and `UniRateBackend` (in code; SHOP use unconfirmed) are shipped. |
| **Conversion** | `convert_money(value, currency)` in `djmoney.contrib.exchange.models`; requires the exchange app to be in `INSTALLED_APPS`. |
| **Display amount** | Not a first-class concept in this library; formatting is delegated to shop-py-moneyed's `format_money` and locale tools. |
| **Allocation** | Not yet implemented; planned once shop-py-moneyed adds allocation support. |
