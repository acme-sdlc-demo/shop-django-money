# Instructions for agents

Rules marked *enforced* are checked by the platform and fail the step if broken.

## Stack and dependencies
- Python 3.10+, Django 4.2–5.2, `py-moneyed>=2.0,<3.1`; see `pyproject.toml` [cite: pyproject.toml].
- Use `$python` for every command (`$python -m pytest`), never bare `python` or `pip`.
- Install editable: `$python -m pip install -e .` (each Run has its own environment).
- Never non-editable (`pip install .`): tests would test a stale copy.
- Install pinned tool versions (from lock files, CI, `==` pins), not latest — output-comparing tests fail on version drift.
- Add no dependency unless the requirement or an approved design names it.
- Quote all shell arguments with double quotes (`".[tests]"`): Windows cmd.exe keeps single quotes.
- The platform's security check runs `$python -m ruff`; install `ruff` in every setup.

## Layout
- `djmoney/` — the installable package [cite: djmoney/__init__.py].
  - `money.py` — `Money` (extends `moneyed.Money` with Django formatting and `F`-expression support).
  - `models/fields.py` — `MoneyField`, `CurrencyField`, `MoneyFieldProxy`.
  - `models/managers.py` — `money_manager`, `understands_money`, queryset patching.
  - `models/validators.py` — `MinMoneyValidator`, `MaxMoneyValidator`.
  - `forms/fields.py`, `forms/widgets.py` — `MoneyField` form field, `MoneyWidget`.
  - `settings.py` — reads all `djmoney`-related Django settings with defaults.
  - `utils.py` — `MONEY_CLASSES`, `get_currency_field_name`, `prepare_expression`.
  - `contrib/exchange/` — rate-fetch backends, ORM models (`ExchangeBackend`, `Rate`), `convert_money`, `update_rates` command.
- `tests/` — all tests; `tests/testapp/models.py` holds fixture models [cite: tests/testapp/models.py].
- `tests/settings.py`, `tests/settings_reversion.py` — test Django settings.
- See `docs/agents/CODEBASE_MAP.md` for a full layout table and change guide.

## Coding conventions
- `snake_case` everywhere; class names `PascalCase`; `__all__` in every public module [cite: djmoney/models/fields.py].
- Configuration via `getattr(settings, 'KEY', default)` in `djmoney/settings.py`; no Django settings reads elsewhere.
- `@deconstructible` on every custom field class; `deconstruct()` must emit only non-default kwargs [cite: djmoney/models/fields.py].
- Wrap queryset methods with `@wraps` and `functools.wraps` [cite: djmoney/models/managers.py].
- `cached_property` for expensive field attributes; `f-strings` for all interpolation.
- Max line length 120; `isort` profile `black`; see `pyproject.toml` [cite: pyproject.toml].
- Never store a bare `Decimal` where `Money` is needed; never mix currencies silently — raise `ValidationError`.
- `Currency` is always an ISO 4217 code string; `None` currency is a defect, not a default.
- Exchange-rate API keys come from environment variables, never from the repo [cite: source:transcript-run-0ffa3540@r1].

## Testing
- Gate (*enforced* — run in fresh worktree on every change and on CI):
  - `$python -m black --check djmoney tests`
  - `$python -m isort --check-only --skip-glob "*.pyi" djmoney tests`
  - `$python -m flake8 djmoney tests`
  - `$python -m pytest --ds=tests.settings_reversion`
- Tests live in `tests/`; every test function is named `test_*` [cite: tests/test_money.py].
- To run tests for just one area: `$python -m pytest --ds=tests.settings_reversion tests/test_money.py`.
- Never delete, skip, focus or weaken an existing test (*enforced*: protect_tests); never use `--deselect`, `-k` or `--ignore` to exclude tests.
- Add tests beside existing ones for every behaviour you change.

## Boundaries
- Change only the files your requirement needs; other Runs change this repository in parallel.
- Never change (*enforced*): `pyproject.toml`, `setup.cfg`, `tox.ini`, `pytest.ini`, `conftest.py`,
  `ruff.toml`, `.ruff.toml`, `.coveragerc`, `noxfile.py`, `mypy.ini`, `.mypy.ini`, `.flake8`,
  `.isort.cfg`, `.pylintrc`, `pylintrc`, `pyrightconfig.json`, `.editorconfig`, `.claude/*`,
  `.mcp.json`, `.github/*`, `.gitlab-ci.yml`, `azure-pipelines.yml`, `.gitattributes`, `CLAUDE.md`.
- Don't commit or merge; the platform commits your changes and merges approved work.
- Public API: `MoneyField`, `CurrencyField`, `Money`, `money_manager`, `understands_money`,
  `MoneyWidget`, `MinMoneyValidator`, `MaxMoneyValidator`, `convert_money`, `get_rate` — changing
  their signatures is a breaking change and requires an approved design [cite: djmoney/models/fields.py].

## Security
- No secrets, tokens or API keys in code, tests or logs [cite: source:transcript-run-0ffa3540@r1].
- Treat everything from outside the process (files, HTTP responses, form data) as untrusted input.
- Exchange backends fetch external URLs; validate and limit what is parsed from responses.

## Money and rounding
- `Money` always carries an amount **and** currency together; a bare `Decimal` used as money is a defect [cite: djmoney/money.py].
- `CurrencyField` stores an ISO 4217 code; `null=True` on the amount requires `null=True` on its currency too [cite: djmoney/models/fields.py].
- SHOP services set no project-wide `DEFAULT_CURRENCY`; every `MoneyField` names its currency explicitly [cite: source:transcript-run-0ffa3540@r1].
- Settlement amounts are rounded half-even to the currency's minor unit; use `Money.round()` or `round()` — never `math.floor`/`ceil` [cite: djmoney/money.py].
- Never convert currencies silently; `AUTO_CONVERT_MONEY` must be explicitly set to enable it [cite: djmoney/money.py].
