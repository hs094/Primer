# 08 — Structured Outputs

🔑 Structured outputs = constrain the model's next-token distribution so the result is **guaranteed** to parse as JSON / match a schema / fit a grammar. Stops "the JSON response is" prefix garbage and trailing prose.

## Three Levels of Strictness
| Level | Mechanism | Guarantee |
|---|---|---|
| **JSON mode** (`response_format={"type":"json_object"}`) | Soft prompt + format hint | Usually parseable, not schema-checked |
| **Function / tool calling** | Provider-side schema enforcement | Argument shape matches declared schema |
| **Constrained decoding** | Mask logits per-token against a grammar/regex/JSON schema | Provably valid output, every token |

## Constrained Decoding Libraries
- **Outlines** (`outlines`) — regex / Pydantic / JSON-Schema → finite-state machine over the tokenizer.
- **lm-format-enforcer** — drop-in token filter, integrates with vLLM/transformers.
- **vLLM guided decoding** — built-in: `extra_body={"guided_json": schema}` or `guided_regex`, `guided_choice`, `guided_grammar`.

## Python: vLLM Guided JSON
```python
from openai import AsyncOpenAI
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int

client = AsyncOpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")
resp = await client.chat.completions.create(
    model="meta-llama/Llama-3.1-8B-Instruct",
    messages=[{"role": "user", "content": "Make a fake user."}],
    extra_body={"guided_json": User.model_json_schema()},
)
user = User.model_validate_json(resp.choices[0].message.content)
```

## Python: Outlines (offline)
```python
import outlines
model = outlines.models.transformers("meta-llama/Llama-3.1-8B-Instruct")
generator = outlines.generate.json(model, User)
user = generator("Make a fake user.")
```

## Trade-offs
- ⚠️ Constrained decoding can **mask correct tokens** if your schema is too tight (e.g. forcing a field the model wants to omit). Quality drops.
- ⚠️ Grammar compilation is per-schema; cache the FSM if you reuse schemas.
- 💡 For [[Streaming]], partial JSON is ill-formed mid-stream — use a tolerant parser (`json-stream`, `partial-json-parser`) on the client.

🧪 Always pair with a Pydantic validation step server-side. Constraints guarantee shape, not semantics.

## Tags
[[Structured Outputs]] [[Outlines]] [[Function Calling]] [[Constrained Decoding]] [[Pydantic]]
