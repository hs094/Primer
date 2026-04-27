# EX 01 — Validating File Data

🔑 **Key insight:** parse the file with the right stdlib loader, then hand the resulting `dict` (or string) to a Pydantic model — Pydantic does not read files, it validates structure.

## When to reach for this pattern

Configuration files, fixture data, scratch data dumps from other systems — anything that arrives as JSON / CSV / TOML / YAML / XML / INI on disk. If you are parsing **settings**, the docs nudge you toward [[Pydantic CO 17 - Settings Management]] (`pydantic-settings`) instead of rolling your own loader. For everything else, the recipe below is the canonical pattern.

The validating model is reused across every format:

```python
from pydantic import BaseModel, EmailStr, PositiveInt

class Person(BaseModel):
    name: str
    age: PositiveInt
    email: EmailStr
```

See [[Pydantic CO 01 - Models]] and [[Pydantic CO 05 - Types]] for `EmailStr` / `PositiveInt`.

## JSON

`model_validate_json` skips the `json.loads` round-trip and validates straight from the string:

```python
import pathlib
from pydantic import BaseModel, EmailStr, PositiveInt

class Person(BaseModel):
    name: str
    age: PositiveInt
    email: EmailStr

json_string = pathlib.Path('person.json').read_text()
person = Person.model_validate_json(json_string)
```

For a list of objects, use [[Pydantic CO 14 - Type Adapter]]:

```python
from pydantic import TypeAdapter

person_list_adapter = TypeAdapter(list[Person])
json_string = pathlib.Path('people.json').read_text()
people = person_list_adapter.validate_json(json_string)
```

## JSON Lines

One JSON object per line — split, then validate each line individually:

```python
json_lines = pathlib.Path('people.jsonl').read_text().splitlines()
people = [Person.model_validate_json(line) for line in json_lines]
```

## CSV

`csv.DictReader` already produces `dict` rows, so feed them straight into `model_validate`:

```python
import csv

with open('people.csv') as f:
    reader = csv.DictReader(f)
    people = [Person.model_validate(row) for row in reader]
```

⚠️ Every CSV cell is a string. Pydantic's coercion handles `"42"` → `int`, but if you have stricter modes configured (see [[Pydantic CO 10 - Validators]]) you may need to widen the field types or relax strict mode.

## TOML

`tomllib` is stdlib in 3.11+; open in **binary** mode:

```python
import tomllib

with open('person.toml', 'rb') as f:
    data = tomllib.load(f)
person = Person.model_validate(data)
```

## YAML

`yaml` is third-party (`PyYAML`). Always use `safe_load` — never `load` — on untrusted input:

```python
import yaml

with open('person.yaml') as f:
    data = yaml.safe_load(f)
person = Person.model_validate(data)
```

## XML

`xml.etree.ElementTree` gives you an element tree; flatten the children into a `dict` then validate:

```python
import xml.etree.ElementTree as ET

tree = ET.parse('person.xml').getroot()
data = {child.tag: child.text for child in tree}
person = Person.model_validate(data)
```

⚠️ This naive flattening only works for one level of `<tag>text</tag>` children. Nested or attribute-heavy XML needs a real adapter.

## INI

`configparser` returns section proxies that behave like dicts:

```python
import configparser

config = configparser.ConfigParser()
config.read('person.ini')
person = Person.model_validate(config['PERSON'])
```

## Pattern recap

| Format | Loader | Pydantic call |
|---|---|---|
| JSON | `pathlib.read_text()` | `model_validate_json` |
| JSONL | split lines | `model_validate_json` per line |
| CSV | `csv.DictReader` | `model_validate` per row |
| TOML | `tomllib.load` (binary) | `model_validate` |
| YAML | `yaml.safe_load` | `model_validate` |
| XML | `ElementTree` | `model_validate` after flattening |
| INI | `configparser` | `model_validate` on section |

Failures surface as `ValidationError` regardless of source — see [[Pydantic ER 02 - Validation Errors]] for unpacking them.

💡 **Takeaway:** Pydantic is format-agnostic — pick the right parser, then let the model do the schema work.
