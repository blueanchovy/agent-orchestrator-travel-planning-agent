
### About

An agent orchestration system that processes user requests through a chain of specialized agents. Each agent performs a specific task and passes context to the next agent in the chain. The orchestrator  manages the conversation flow, handles errors, and supports agent collaboration.

### Scenario

A Travel Planning Assistant that uses multiple specialized agents working together to:

Understand natural language travel requests(Parser)
Extract and validate required information(Validator)
Fetch travel options from multiple providers in parallel, via mock API calls (Providers)
Compare and select best options
Return a formatted response to the user(Collection Point + Aggregator + Formatter)

### Architecture

The workflow is in three phases, with a conditional exit after validation:

Workflow Table

| Phase | Work | Why it has this execution policy |
| --- | --- | --- |
| 1. Prepare — sequential | Parser, then validator | Validation needs the parser’s structured fields. If required information is missing, provider work must not start. |
| 2. Search — concurrent | Provider A, B, and C | Each receives the same validated request and performs an independent lookup. |
| 3. Resolve — sequential | Collect outcomes, aggregate, then format | The decision needs the provider outcomes; the response needs the decision. |


The key dependency is simple: a provider cannot search until it has a usable travel request. But once the request is usable, Provider A does not need Provider B’s result to do its own search. The orchestration is sequential around one concurrent group. 

Dependency Table 

| Node | Required input | Possible next work |
| --- | --- | --- |
| Parser | User message | Validator |
| Validator | Parsed request | All three providers if valid; formatter if clarification is needed |
| Each provider | The same validated context | Collection point |
| Collection point | One outcome from each provider: result, error, or timeout | Aggregator |
| Aggregator | Collected outcomes | Formatter |
| Formatter | Clarification or aggregation decision | User-facing response |

On the clarification branch, Phase 2 and aggregation are skipped. The formatter can still turn “missing destination and travel type” into a useful question for the user. This is a conditional path, not a fourth execution phase.

There are seven agent instances: one parser, one validator, three providers, one aggregator, and one formatter. The three providers can be instances of the same configurable provider class; they need not be three separate implementations. The orchestrator is the coordinator that invokes these agents and applies the branching rule, rather than another travel-specialist agent.

Concurrency does not automatically provide failure isolation. The orchestrator must choose a policy that preserves successful outcomes when another provider raises an exception or exceeds its timeout. The minimum assessment-focused policy is best effort: do not retry; collect each provider’s outcome, continue with available options, and report reduced coverage.

The collection point keeps one identified outcome per provider. The aggregator then applies an explicit rule—say, select the cheapest successful option—and decides how complete the search was. There being three configured providers, the graph supports all four counts:

Provider Search Outcomes

| Successful providers | Aggregation decision | User-facing implication |
| --- | --- | --- |
| 3 | Full success | Compare all three options. |
| 2 | Partial success | Compare two; note that coverage is incomplete. |
| 1 | Partial success | Show the one available option; note that coverage is incomplete. |
| 0 | Total failure | Do not claim to have found a “best” option; explain that no provider returned an option. |