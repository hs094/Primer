# HT 01 — General How To

🔑 The "How To" section is a recipe book — short, focused snippets for tasks the tutorial deliberately doesn't cover. Read them when you hit a specific need; not front-to-back.

## When to use a recipe

- You already know the basics ([[FastAPI 01 - First Steps]]).
- You hit a *specific* requirement (custom OpenAPI? GraphQL? alternative routing?).
- The tutorial answer would be "yes, you can — here's the small twist".

## Recipe inventory

- [[FastAPI HT 02 - Migrate Pydantic v1 to v2]] — moving an old codebase forward.
- [[FastAPI HT 03 - GraphQL]] — Strawberry / Ariadne integration.
- [[FastAPI HT 04 - Custom Request and APIRoute]] — middleware-like behavior, scoped per route group.
- [[FastAPI HT 05 - Conditional OpenAPI]] — hide docs in prod.
- [[FastAPI HT 06 - Extending OpenAPI]] — patch the schema.
- [[FastAPI HT 07 - Separate OpenAPI Schemas Input Output]] — when input ≠ output.
- [[FastAPI HT 08 - Custom Docs UI Static Assets]] — self-host Swagger.
- [[FastAPI HT 09 - Configure Swagger UI]] — tweak the UI.
- [[FastAPI HT 10 - Testing a Database]] — patterns for DB-backed tests.
- [[FastAPI HT 11 - Use Old 403 Auth Status Codes]] — back-compat for ancient clients.

## Mental model

Tutorial = the path. Advanced = the diversions. How-To = the toolbox. Reach for a recipe when "the right way" is non-obvious or framework-specific.

💡 If you find yourself reinventing one of these recipes, the docs probably already have a one-liner — search before you build.
