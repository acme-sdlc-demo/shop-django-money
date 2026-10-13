# Open questions

## Open questions

**Q1. Does the form-layer currency rejection message name the offending currency?**
The product note requires "Form validation that rejects an amount in a currency the field doesn't
allow, with a message naming the currency." [cite: source:product-note@r1]
The form-layer `MoneyField` uses Django's `ChoiceField` for the currency widget, which raises a
generic "Select a valid choice: %(value)s is not one of the available choices" error — the currency
code does appear in that message, but this is not a purpose-built message. Whether this meets the
product requirement needs a decision from the product owner or tech lead.
[cite: C:\acc26\r1\work\worktrees\run-0ffa3540\djmoney\forms\fields.py]

**Q2. Is shop-py-moneyed's allocation feature available yet?**
The product note says the admin allocation display (split bills, refunds across order lines) is
planned "once the money library can allocate." [cite: source:product-note@r1]
It is unclear whether shop-py-moneyed (pinned to `py-moneyed>=2.0,<3.1`) has shipped allocation
support yet. A person needs to check the current shop-py-moneyed release and determine whether the
admin work can begin.
[cite: C:\acc26\r1\work\worktrees\run-0ffa3540\pyproject.toml]

**Q3. Does SHOP configure a no-op Django cache, or does the rate cache actually operate?**
Carol stated "No cache layer" for exchange rates [cite: source:transcript-run-0ffa3540@r1], but the
code in `get_rate()` uses `cache.get` / `cache.set` with a default timeout of 600 s
(`RATES_CACHE_TIMEOUT`). [cite: C:\acc26\r1\work\worktrees\run-0ffa3540\djmoney\contrib\exchange\models.py]
[cite: C:\acc26\r1\work\worktrees\run-0ffa3540\djmoney\settings.py]
If SHOP services use Django's default `LocMemCache` the cache is in-process only and may be
acceptable. If they use a null cache backend it is effectively disabled. A person needs to confirm
which Django cache backend SHOP services configure, and whether the 600 s default is appropriate.

**Q4. Is there a third rate provider (UniRate) in scope for SHOP?**
The code ships a `UniRateBackend` that calls `unirateapi.com` over HTTP, and the settings file defines
`UNIRATE_URL` and `UNIRATE_ACCESS_KEY`.
[cite: djmoney/contrib/exchange/backends/unirateapi.py]
[cite: C:\acc26\r1\work\worktrees\run-0ffa3540\djmoney\settings.py]
The product note and the interview mention only Open Exchange Rates (primary) and Fixer (fallback).
[cite: source:product-note@r1] [cite: source:transcript-run-0ffa3540@r1]
A person needs to confirm whether UniRate is a planned third provider for SHOP or an upstream
addition that SHOP does not use.

**Q5. What happens when a `MoneyField` has no `default_currency` and no numeric default?**
The `DEFAULT_CURRENCY` setting is `None` by default and SHOP sets none. A `MoneyField` with a
numeric `default=` but no `default_currency=` triggers an `AssertionError` at class-definition time.
A `MoneyField` with no `default=` at all is valid. Whether the library should raise a cleaner error
(e.g. `ImproperlyConfigured`) or emit a system check warning has not been decided.
[cite: C:\acc26\r1\work\worktrees\run-0ffa3540\djmoney\models\fields.py]
[cite: source:transcript-run-0ffa3540@r1]

**Q6. What does `deconstruct()` omit and why — comments are marked as needed in the code.**
`deconstruct()` conditionally omits `default_currency` and `currency_choices` from the serialised
field state, but both conditionals are marked "Comment needed: Why is this done?"
[cite: C:\acc26\r1\work\worktrees\run-0ffa3540\djmoney\models\fields.py]
The intent appears to be migration stability (only emit a kwarg when it differs from the global
default), but the exact reasoning has not been recorded. A person should document and verify the
condition for each kwarg.

## Documents vs code

**Discrepancy 1: "No cache layer" vs. active rate cache in the code.**
Carol stated in the interview that the exchange-rate storage has "No cache layer."
[cite: source:transcript-run-0ffa3540@r1]
The code in `djmoney/contrib/exchange/models.py` (`get_rate()`, lines 49–53) calls
`cache.get(key)` before hitting the database and `cache.set(key, result, RATES_CACHE_TIMEOUT)` after,
where `RATES_CACHE_TIMEOUT` defaults to 600 s.
[cite: C:\acc26\r1\work\worktrees\run-0ffa3540\djmoney\contrib\exchange\models.py]
[cite: C:\acc26\r1\work\worktrees\run-0ffa3540\djmoney\settings.py]
Whether Carol meant that SHOP configures Django's cache to a no-op backend, or was simply unaware
of the in-process cache, is not resolved.

**None found (release process):** The release process described in the document matches the code
repository structure — no PyPI tooling, version sourced from `djmoney.__version__` via
`[tool.setuptools.dynamic]`, releases by tag.
[cite: source:how-we-release@r1]
[cite: C:\acc26\r1\work\worktrees\run-0ffa3540\pyproject.toml]
