# 02 — Subjects

🔑 Subjects are dot-separated hierarchical names (`orders.created.us`) — the only routing key NATS knows.

Source: https://docs.nats.io/nats-concepts/subjects

## Anatomy
- Tokens joined with `.` form a hierarchy: `time.us.east.atlanta`.
- Allowed chars: alphanumeric, `_`, `-`. Avoid spaces and unicode.
- Reserved: `.` (separator), `*` (single-token wildcard), `>` (multi-token tail), leading `$` (system).

## Wildcards
| Pattern | Matches | Doesn't match |
|---|---|---|
| `time.*.east` | `time.us.east`, `time.eu.east` | `time.us.east.atlanta` |
| `time.us.>` | `time.us.east`, `time.us.east.atlanta` | `time.eu.east` |
| `time.>` | anything under `time.` | `time` (must have ≥1 trailing token) |

⚠️ `*` matches **exactly one** token; `>` matches **one or more** and must be the **last** token.

## Naming Conventions
- 💡 Encode business intent, not transport details: `orders.created.us`, not `tcp.json.orders`.
- Keep depth ≤ 16 tokens; deeper trees hurt subscriber matching perf.
- Prefer `noun.verb.qualifier` (`account.signup.success`) over `verb.noun` for natural wildcard groupings.
- 🧪 Subscribers can fan out via wildcards; publishers always use a concrete subject.

## Worked Examples
```
orders.created.us           → publisher
orders.created.>            → regional analytics (US, EU, APAC)
orders.*.us                 → all US-only events
orders.>                    → audit log
```

## Tags
[[NATS]] [[Pub Sub]] [[Messaging]]
