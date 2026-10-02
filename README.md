# LLM Message Array Linter

Paste the Anthropic `messages` array from a request that returned a 400 and get the exact block index that broke it, plus every index where the transcript can legally be truncated.

**Live demo:** https://0xelitesystem.github.io/llm-message-array-linter/

## Use

1. Paste the `messages` array (JSON) from the failing request, or pick a sample.
2. Click Analyze.
3. Read the findings, each anchored to a path such as `messages[1].content[3]`.
4. Use the truncation map, or enter a Keep last value and click Find nearest legal cut, then copy findings or legal cut indices.

## Why this exists

A 400 from the Messages API rarely says which block broke the request, and a transcript is full of customer data you should not paste into a hosted validator. This is one HTML file with no tracking and no network calls that finds the breaking block and the safe truncation points locally, MIT licensed.

## Features

- **Names the breaking block.** Findings are anchored to a concrete path such as `messages[1].content[3]`, not to a message number.
- **Walks spans, not messages.** The atomic unit of a tool round trip crosses a message boundary by definition: an assistant `tool_use` and the user `tool_result` that answers it are one unit. The linter models that unit directly.
- **Catches the partially answered parallel fan-out.** Four `tool_use` blocks, three `tool_result` blocks: every individual block is well-formed, so a naive per-block validator passes and the request 400s anyway. This is the case the tool exists for.
- **Truncation map.** For every candidate new start index it reports whether the cut is span-legal, so context-trimming code stops manufacturing new 400s. A "keep last N messages" helper snaps a requested cut back to the nearest legal one.
- **Orphan detection in both directions.** `tool_use` with no result, and `tool_result` whose `tool_use_id` matches nothing (the classic symptom of a trimmer that dropped the assistant turn and kept the user turn answering it).
- **Placement and ordering.** Results that are not in the message immediately after their call, results split across several user messages, and text placed before `tool_result` in the same user message.
- **Thinking-block checks.** Flags reconstructed or unsigned `thinking` / `redacted_thinking` blocks on the most recent assistant message, which is an API rule returning `400 invalid_request_error`.
- **Dated receipts, kept separate.** Every rule ships with a verbatim vendor quote, its source URL, and the date it was checked, in a section that is visually separated from the computation so a stale quote can never make correct logic look wrong.
- Zero external dependencies, no build step, no API key, no network request. One HTML file.

## How it works

A **span** opens at an assistant message containing one or more `tool_use` blocks and closes at the user message carrying a `tool_result` for every one of those ids. The linter indexes every block, matches ids in both directions, then evaluates each span for completeness, placement, and ordering.

The truncation map falls out of the same index. A cut at index `i` keeps `messages[i..end]`, and it is **span-legal** exactly when no surviving `tool_result` references a `tool_use` that the cut would drop. That single condition is what makes a cut inside a parallel fan-out illegal: the fan-out's results all point back at one message.

Three notes on scope, because each one is a place where a confident-sounding rule turns out not to hold.

**The user-role head rule is advisory, not a verdict.** The claim that a trimmed conversation must begin with a `user` turn appears in no current primary Anthropic source. Checked 2026-08-09: it is not in the Messages API `messages` parameter description, not on the errors page, and not in the Amazon Bedrock mirror. What does exist is a runtime error string, `messages: first message must use the "user" role`, pasted into a public bug report on 2024-03-16 with HTTP 400 and `invalid_request_error`. The truncation map therefore reports head role in its own advisory column and never lets it decide SAFE or UNSAFE. Note also that this and the documented merge behaviour are orthogonal, not contradictory: "Consecutive `user` or `assistant` turns in your request will be combined into a single turn" is about collapsing adjacent same-role turns and says nothing about which role may lead.

**The thinking-block rule is stronger than usually stated, and narrower.** It is an API rule returning `400 invalid_request_error`, not an SDK convention. It binds on the **most recent** assistant message, not on every assistant turn. And it is not conditional on opting into extended thinking: on several current models thinking is on with no configuration, so the blocks arrive whether or not you asked for them. Whether *prior* turns' thinking stays in context is model-dependent, which is why this tool never tells you to hand-prune it: on keep-all models those blocks remain in context and are billed as input tokens, and on last-turn-only models the API strips them for you.

**Anthropic only, deliberately.** There is no cross-vendor shape auto-detect. OpenAI's Chat Completions uses separate `role: "tool"` messages and the Responses API uses flat `function_call_output` items keyed by `call_id`; neither carries the fan-out atomicity constraint that this tool is built around, so pretending one linter covers both would be a false claim about someone else's API.

The core is a set of pure functions: `parseMessagesInput(text)`, `normalizeMessages(messages)`, `analyzeTranscript(messages)`, `buildTruncationMap(rows, toolUseById, toolResultsById)`, and `nearestLegalCut(points, desiredStart)`. Each takes input and returns a result object, touches no DOM, and can be lifted out of the page and run in Node.

## Privacy

Everything runs in your browser. The transcript you paste is never uploaded, logged, or transmitted; the page makes no network requests at all and loads no third-party resources. There is no API key, no backend, and no analytics. Transcripts routinely contain customer data and tool output from internal systems, which is exactly why this is a single static file you can read top to bottom, or save and open offline.

The only thing stored is your light or dark theme preference, in `localStorage`. The vendor source links in the page open external sites only when you click them.

## Run locally

```
git clone https://github.com/0xelitesystem/llm-message-array-linter
cd llm-message-array-linter
```

Open `index.html` in any modern browser. Or serve the folder with `python -m http.server` and visit http://localhost:8000.

## Build

No build step. The whole tool is one `index.html` with inline CSS and JavaScript and no dependencies.

## License

MIT. See [LICENSE](LICENSE).

## More

- More tools: https://0xelitesystem.github.io/
- Built by elitesystem.ai: https://elitesystem.ai

Companion tools that also take a real request body as input: [Prompt Cache Inspector](https://0xelitesystem.github.io/prompt-cache-inspector/) finds where a cached prefix broke, and [LLM Stream Inspector](https://0xelitesystem.github.io/llm-stream-inspector/) diagnoses a raw SSE dump.
