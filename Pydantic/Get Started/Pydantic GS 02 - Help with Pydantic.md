# GS 02 — Help with Pydantic

🔑 **Key insight:** Read the docs first, then ask on GitHub Discussions — Stack Overflow and YouTube fill in the long tail.

## Read the docs

- **Usage docs** — start at [[Pydantic CO 01 - Models]] and walk forward through the Concepts section.
- **API reference** — the canonical signatures live under the API docs (`pydantic.BaseModel`, `pydantic.Field`, `pydantic.TypeAdapter`, …).

If your question is "how do I X?", the answer is almost always one search away in the docs before it's worth pinging humans.

## Ask the community

### GitHub Discussions

> GitHub discussions are useful for asking questions, your question and the answer will help everyone.

- URL: <https://github.com/pydantic/pydantic/discussions>
- Best for: design questions, "is this a bug?", "is there an idiomatic way to…?"
- Public, searchable, and seen by the core team.

### Stack Overflow

- Tag: [`pydantic`](https://stackoverflow.com/questions/tagged/pydantic)
- Best for: standalone, reproducible questions with a clear right answer.

⚠️ Per the docs: the `pydantic` tag is *not always monitored by the core Pydantic team* — answers come from the wider community, so quality varies.

### YouTube

> YouTube has lots of useful videos on Pydantic.

- Search: <https://www.youtube.com/results?search_query=pydantic>
- Featured: Marcelo Trylesinski's "Pydantic V1 to V2 — The Migration" — worth a watch if you're stuck on [[Pydantic GS 04 - Migration Guide]].

## Reporting a bug

- Open an issue on the GitHub repo with a minimal reproducer.
- Include: Pydantic version (`pydantic.version.VERSION`), Python version, and the smallest model + input that triggers the problem.
- For validation-specific surprises, also paste the `ValidationError` output — see [[Pydantic ER 01 - Error Handling]] for what's in there.
- Mention which platform (Linux/macOS/Windows) — relevant for `tzdata`, regex, and Rust-core surprises.

```python
import pydantic
print(pydantic.VERSION)
```

## Asking well

- Show the code you actually ran, not a paraphrase.
- Show the full traceback or `ValidationError`.
- State expected vs actual behavior in one sentence each.
- Mention major-version (V1 vs V2) — most stale advice on the internet is V1.
- If type-checker output is involved, name the checker and version (mypy, pyright, etc.) — Pydantic ships its own mypy plugin, see [[Pydantic GS 04 - Migration Guide]].

## Adjacent communities

The Pydantic team maintains several sibling packages — file bugs in their *own* repos, not the main `pydantic` one:

- `pydantic-core` — the Rust validation core. Bugs in coercion, performance regressions, or panics belong here. See [[Pydantic IT 01 - Architecture]].
- `pydantic-settings` — `BaseSettings` and env loading. See [[Pydantic CO 17 - Settings Management]].
- `pydantic-extra-types` — `Color`, `PaymentCardNumber`, country/phone types.

For FastAPI-specific Pydantic questions, the FastAPI repo and Discord are usually faster than asking the Pydantic team. Same for LangChain, Django Ninja, and other downstream consumers — start with the framework's channel before escalating to Pydantic itself.

## Quick triage checklist before you ask

1. Are you on V1 or V2? Run `python -c "import pydantic; print(pydantic.VERSION)"`.
2. Did the docs for *this* major version address it? Most stale Stack Overflow answers are V1 — see [[Pydantic GS 04 - Migration Guide]].
3. Can you reproduce with the smallest possible model and a hard-coded input dict?
4. Have you read the `ValidationError` `loc` and `type` fields, not just the `msg`? See [[Pydantic ER 01 - Error Handling]].
5. Is it really a Pydantic question? `BaseSettings` lives in `pydantic-settings` ([[Pydantic CO 17 - Settings Management]]); `Color`/`PaymentCardNumber` in `pydantic-extra-types`.

## Things the docs are usually best at answering

- "Why does my `Optional[X]` field not default to `None`?" → [[Pydantic GS 04 - Migration Guide]].
- "How do I customize JSON output?" → [[Pydantic CO 09 - Serialization]].
- "Strict vs lax — which one runs when?" → [[Pydantic CO 13 - Strict Mode]].
- "How do I validate a `list[Foo]` without wrapping in a model?" → [[Pydantic CO 14 - Type Adapter]].

💡 **Takeaway:** Search the docs, then GitHub Discussions, then Stack Overflow — in that order — and always paste a minimal reproducer with your Pydantic version pinned.
