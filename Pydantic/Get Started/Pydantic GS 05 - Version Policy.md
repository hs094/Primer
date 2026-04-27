# GS 05 — Version Policy

🔑 **Key insight:** No intentional breaking changes inside V2 — deprecated APIs survive until V3, and major versions are aimed for roughly once a year.

## V2 stability promise

> We will not intentionally make breaking changes in minor releases of V2.

Anything deprecated in a V2 minor release stays callable until V3 ships. Pydantic targets roughly **annual major releases** going forward, but with breaking-change budgets far smaller than the V1→V2 jump.

## What does *not* count as a breaking change

These are explicitly fair game in a V2 minor release — pin or test against them if your code depends on the exact behavior:

- Bug fixes that affect undocumented features or constructs.
- Adjustments to JSON Schema reference (`$ref`) format.
- Changes to `ValidationError` exception fields — `msg`, `ctx`, `loc` (see [[Pydantic ER 01 - Error Handling]]).
- New keys appearing inside a validation error dict.
- `__repr__` output changes on models, fields, etc.
- Internal core-schema content changes (the structure returned by `__get_pydantic_core_schema__`).

⚠️ If you're parsing `ValidationError.errors()` output, treat the schema as additive-only — new keys can show up in any minor release.

## Deprecation policy

- Deprecations are announced in release notes and emit `DeprecationWarning` (or `PydanticDeprecatedSince*`).
- Deprecated APIs remain functional until the next major bump.
- Migration paths are spelled out in the warning message and the docs.

## Python version support

Pydantic drops a Python version when **either**:

1. That Python version reaches its official end-of-life, **or**
2. Less than **5%** of recent minor-release downloads use that version.

Currently V2 supports Python **3.9+** (see [[Pydantic GS 03 - Installation]]). When 3.9 falls below the 5% threshold or hits EOL, expect it to be dropped in the next minor release.

## Experimental features

New, in-development features land either:

- Inside the `pydantic.experimental` module, or
- With an `experimental_` prefix on the public name.

⚠️ These can change shape or be removed with limited notice — they are *not* covered by the V2 stability promise. Don't ship anything load-bearing on top of them. Catalog in [[Pydantic CO 19 - Experimental]].

## Pinning recommendations

For library code:

```toml
# pyproject.toml — libraries: pin to the major
dependencies = ["pydantic>=2,<3"]
```

For applications, pin tighter to control upgrades:

```toml
# pyproject.toml — apps: minor-pin and bump intentionally
dependencies = ["pydantic>=2.x,<2.(x+1)"]
```

⚠️ The `pydantic.v1` compat namespace is for migration, not long-term reliance — it tracks V1 1.10 and won't get new features. Plan to finish migrating before V3 ships and removes it.

## What this means in practice

- Upgrade across V2 minors freely; review release notes for deprecation warnings before each bump.
- Don't pattern-match on exact validation-error `msg` strings — match on `type` instead, since `msg` can change in any minor release.
- Treat `experimental_` names like preview APIs: fine to play with, not fine to ship.
- Watch the EOL / 5% usage thresholds when picking your minimum supported Python.
- If you parse `model_json_schema()` output downstream, expect `$ref` formatting tweaks across minors — code defensively.

## Cross-references

- Coming from V1 and worried about churn → [[Pydantic GS 04 - Migration Guide]].
- Surface area covered by the promise → core schema in [[Pydantic IT 01 - Architecture]].
- Error-shape stability rules → [[Pydantic ER 01 - Error Handling]].

💡 **Takeaway:** Pin to a major, watch the deprecation warnings, and assume `experimental_` features can vanish — everything else in V2 is safe to lean on.
