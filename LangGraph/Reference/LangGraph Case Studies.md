# LangGraph Case Studies

🔑 LangGraph powers production agents across finance, healthcare, code generation, customer support, and search at companies like Uber, LinkedIn, Replit, Klarna, BlackRock, and J.P. Morgan — the case-study list is the fastest way to map "what's actually shipping" to use-case archetypes.

## What This Page Is

> This list of companies using LangGraph and their success stories is compiled from public sources. If your company uses LangGraph, we'd love for you to share your story and add it to the list. You're also welcome to contribute updates based on publicly available information from other companies, such as blog posts or press releases.

A curated, link-out registry — every row is `Company × Industry × Use case × Reference`. Treat it as a primary source for "is this pattern de-risked yet?" rather than a tutorial.

## Use-Case Archetypes Observed

The "Use case" column collapses into a handful of recurring shapes:

- **Copilot for domain-specific task** — AppFolio, BlackRock, Definely, Elastic, Infor, J.P. Morgan, Klarna, Komodo Health, OpenRecovery, Rakuten, Rexera, Tradestack, Unify, Vizient
- **Code generation** — GitLab, Inconvo, LinkedIn, Qodo, Replit, Uber, Vodafone
- **Customer support** — Cisco CX, Cisco TAC, Infor, Minimal, Prosper
- **Search / discovery** — Exa, Harmonic, LinkedIn, Abu Dhabi Government, Vodafone
- **Research & summarization** — Athena Intelligence, Morningstar, 11x
- **Data extraction** — Captide, WebToon
- **GenAI embedded product experiences** — Docent Pro, Infor, Modern Treasury, Monday, Pigment
- **Browser / DevOps automation** — AirTop, C.H. Robinson, Cisco Outshift
- **Developer productivity** — Uber

## Notable Production Deployments

### Code-generation agents

- **Replit** — agentic coding inside the Replit IDE; documented in the [Replit case study, 2024](https://blog.langchain.dev/customers-replit/) and the [Breakout agent story](https://www.langchain.com/breakoutagents/replit).
- **Uber** — developer productivity + code generation across the org; see the [Interrupt 2025 talk](https://youtu.be/Bugs0dVcNI8?feature=shared).
- **LinkedIn** — code generation plus internal "text-to-SQL for data analytics"; covered in the [LinkedIn engineering blog, 2025](https://www.linkedin.com/blog/engineering/ai/practical-text-to-sql-for-data-analytics).
- **GitLab** — Duo Workflow agentic features; see the [Duo workflow design doc](https://handbook.gitlab.com/handbook/engineering/architecture/design-documents/duo_workflow/).
- **Qodo** — explicitly chose LangGraph for their coding agent; see [Why we chose LangGraph to build our coding agent](https://www.qodo.ai/blog/why-we-chose-langgraph-to-build-our-coding-agent/).

### Financial services & fintech

- **BlackRock** — domain copilot, [Interrupt 2025 talk](https://youtu.be/oyqeCHFM5U4?feature=shared).
- **J.P. Morgan** — domain copilot, [Interrupt 2025 talk](https://youtu.be/yMalr0jiOAc?feature=shared).
- **Klarna** — copilot use case, [Klarna case study, 2025](https://blog.langchain.dev/customers-klarna/).
- **Morningstar** — research & summarization, [video story, 2025](https://youtu.be/6LidoFXCJPs?feature=shared).
- **Modern Treasury, Pigment, Prosper** — embedded GenAI / customer support across fintech.

### Customer support

- **Cisco TAC** and **Cisco CX** — both run LangGraph-backed support flows.
- **Minimal** — multi-agent customer support, [How Minimal built a multi-agent customer support system with LangGraph + LangSmith](https://blog.langchain.dev/how-minimal-built-a-multi-agent-customer-support-system-with-langgraph-langsmith/).

### Healthcare & non-profit

- **Komodo Health**, **OpenRecovery**, **Vizient** — domain copilots.
- **City of Hope** — non-profit copilot, [video story, 2025](https://youtu.be/9ABwtK2gIZU?feature=shared).

### Search-heavy products

- **Exa**, **Harmonic** — GenAI-native search engines built on LangGraph.
- **Abu Dhabi Government** ([TAMM](https://www.tamm.abudhabi/)) — government-scale search, [case study, 2025](https://blog.langchain.com/customers-abu-dhabi-government/).

## How To Use This List

⚠️ The page is a directory, not a cookbook — there are no code samples on it. Two practical workflows:

1. **De-risking a design decision.** Find a row with a similar `Industry × Use case` and read the linked case study / talk before committing to an architecture. The Interrupt talks (BlackRock, J.P. Morgan, Uber, LinkedIn, 11x, Unify) are dense with production constraints.
2. **Pattern mining.** Cross-reference rows against the LangGraph patterns in this vault — most case studies map cleanly onto [[LangGraph CO 01 - Workflows and Agents]] (router, orchestrator-worker, evaluator-optimizer) and the streaming / interrupts / time-travel primitives.

## Contributing

> If your company uses LangGraph, we'd love for you to share your story and add it to the list. You're also welcome to contribute updates based on publicly available information from other companies, such as blog posts or press releases.

The page is maintained at [`src/oss/langgraph/case-studies.mdx`](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/case-studies.mdx) — submit a PR or file an issue.

💡 When evaluating LangGraph for a new system, scan this list for one direct analog (same industry) and one structural analog (same use-case archetype, different industry) — the combination usually surfaces both the domain gotchas and the agent-shape gotchas before you write a line of code.
