# Agent Note: Skip MCP tools whose input schema is outside the harness JSON Schema subset

Status: implemented

English | [中文](2026-08-16-mcp-client-input-schema-containment.zh.md)

## Problem

The [MCP client](../feature/2026-07-07-mcp-client-plugin.md) passes each tool's `inputSchema` straight into the registered `ToolDefinition.parameters`. The enforced JSON Schema subset (`assertSupportedJsonSchema`) is applied to the tool's **output** schema — both at registration and through the bridge's `supportedOutputSchema` fallback — but never to the **input** schema.

The DeepSeek provider sends `parameters` verbatim in every request, and the API rejects the whole request when any one function schema is invalid. One malformed tool therefore broke every turn for every tool of that server, not just the malformed one. The observed case was an OpenAPI→MCP gateway emitting `"type": "MultipartFile"` for a multipart file parameter, plus several `enum` keywords on `type: "array"` nodes; all three are outside the subset.

## Decision

In the fetch phase of `syncTools` (`packages/mcp/mcp-client/src/tools.ts`), each tool's `inputSchema` is checked with `assertSupportedJsonSchema` before its definition is built. A tool whose schema is outside the subset is skipped with a `ctx.logger.warn` naming the tool; the server's remaining tools register normally. The check runs in the fetch phase, before the swap, so a skipped tool never reaches the register loop and a re-sync that drops a tool is a plain generation replacement.

This is the input-side mirror of `supportedOutputSchema`, except it skips instead of degrades: an output schema falls back to unconstrained `JsonValue` without losing anything the caller relied on, but the input schema **is** the argument contract the model reads to construct a call, so there is no lossless "unconstrained" replacement — the only containment that does not rewrite the schema is not to register the tool.

No new configuration is added. The output path has no switch (it always contains), and no consumer has asked for a strict "fail the server" mode; the pre-fix behavior — register the malformed tool and fail every turn — is the defect, not a mode worth preserving behind a default.

The register loop keeps its all-or-nothing rollback. Schema-invalid tools are filtered in the fetch phase, so the only failure that can still reach `ctx.tools.register` is a genuine name conflict — a foreign registration squatting on the `mcp__<serverName>__` namespace — which still rolls back loudly, exactly as the original plugin note requires.

## Alternatives considered

**Degrade the input schema to an empty object and keep the tool.** Rejected: unlike output, where falling back to unconstrained `JsonValue` loses nothing the caller relied on, the input schema is the argument shape the model reads to construct a call. Replacing it with an empty object hides every argument and produces calls the server must reject one by one; skipping is the honest containment because the model never sees a tool it cannot call correctly.

**Add an `invalidToolSchema: "skip" | "fail"` config key.** Rejected as unbacked configurability: the output path has no such switch and always contains, and no consumer has requested strict failure. The pre-fix "fail" is the bug, not a mode worth preserving; a strict mode can be added later with evidence.

**Wrap each `ctx.tools.register` in its own try/catch and skip on failure.** Rejected: schema-invalid tools are already filtered in the fetch phase, and the register loop's rollback-to-zero on a conflict is a deliberate invariant (never a partial set). Letting a conflict skip the tool would silently drop a registration under a foreign squatter instead of surfacing it.

**Sanitize the invalid schema, e.g. replace `"MultipartFile"` with `string`.** Rejected: the bridge does not own MCP schema semantics and cannot guess the correct type. This is the heuristic/sanitize line the original plugin note already draws.

## Testing

`packages/mcp/mcp-client/tests/mcp-client.spec.ts` pins the skip: a server listing one tool with a nested `"MultipartFile"` type and one valid tool registers only the valid one, leaves the malformed name unregistered, and warns. Existing cases already pin that parameterless object-rooted schemas (`{"type":"object"}` and `{"type":"object","properties":{}}`) register, so the check cannot regress into dropping no-argument tools.

## Consequences

One malformed input schema no longer breaks the whole server or any turn; it costs only that tool, surfaced by a warn. The available-tool set now depends on schema validity — a pure function of the server's own tool list — rather than on load order or cross-server collisions, so the original "never silently skipped / never load-order-dependent" invariant keeps its scope: the skip is loud and deterministic. The bridge still never rewrites a schema; it only refuses to register one it cannot represent.
