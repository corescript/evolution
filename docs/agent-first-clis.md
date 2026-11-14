# Designing Agent-First Command-Line Interfaces

**A practical engineering guide based on Cloudflare's `cf` CLI and the emerging requirements of agentic software development.**

## Design philosophy

An agent-first CLI is a command-line interface designed so that AI agents can discover, understand, execute, and verify operations without relying on memorized documentation or human intervention.

It should remain useful to human developers, but its primary interface contract is optimized for programmatic consumption.

**The fundamental design principle:**

> Make capabilities discoverable, inputs predictable, outputs structured, and effects verifiable.

A successful agent-first CLI minimizes the amount of reasoning, context, and trial-and-error required to complete a task.

### Core objectives

1. **Discoverability:** An agent can determine which commands exist and what they do.
2. **Consistency:** Similar operations follow the same conventions throughout the CLI.
3. **Structured communication:** Inputs, outputs, and errors can be interpreted without scraping prose.
4. **Context efficiency:** Agents retrieve only the information needed for their current task.
5. **Safety:** Commands make their targets and side effects explicit.
6. **Composability:** Commands can be chained, filtered, and incorporated into larger workflows.
7. **Verifiability:** Agents can establish whether an operation actually succeeded.
8. **Completeness:** The CLI exposes the useful capabilities of the underlying application or platform.

The CLI should be designed as a machine-consumable interface, not merely a human-oriented interface with AI instructions attached.

## Treat schemas as the source of truth

Cloudflare's most consequential architectural decision was generating its new CLI from the same API definitions used to produce other developer interfaces.

Instead of manually maintaining hundreds of commands, it uses its Forge generation system to cover thousands of operations.

### Recommendations

**Maintain one authoritative command schema.**

For every operation, describe:

- Stable operation identifier and command path.
- Human-readable summary and detailed description.
- Arguments, options, types, required fields, and defaults.
- Accepted values, constraints, and validation rules.
- Input and output schemas.
- Whether the operation reads or modifies state.
- Permissions and authentication requirements.
- Whether the operation supports dry runs, local execution, or idempotency.
- Possible error conditions.

For services with HTTP APIs, OpenAPI is a reasonable foundation. Add annotations for CLI-specific semantics that HTTP schemas cannot express.

For local developer tools, a declarative command registry may be more appropriate.

Generate as many interfaces as possible from that shared definition:

- CLI commands and argument parsers.
- CLI help and shell completions.
- Machine-readable command metadata.
- Agent command discovery indexes.
- MCP tool definitions, when needed.
- Documentation and validation schemas.

**Avoid metadata drift.** A flag advertised to an agent must actually work. Defaults shown in machine-readable metadata must match runtime behavior.

Not every operation belongs in generated code. Complex workflows, interactive commands, and local operations may require handwritten implementations, but they should register through the same metadata system.

## Make the CLI self-discovering

An agent should be able to use a CLI that did not exist when its model was trained.

Cloudflare addresses this through natural-language command discovery and structured command inspection.

### Recommended discovery interface

Use a small, consistent set of commands:

| Command | Purpose |
|---|---|
| `mycli --help` | Top-level orientation |
| `mycli cli search "..."` | Natural-language command discovery |
| `mycli schema <command>` | Machine-readable command specification |
| `mycli <command> --help` | Human-readable usage and examples |
| `mycli docs search "..."` | Search supporting documentation |
| `mycli --version` | Identify the installed implementation |

For example, an agent trying to create a database could execute:

1. `mycli cli search "create a database"`
2. `mycli schema databases create`
3. `mycli databases create --name example --dry-run`
4. `mycli databases create --name example`
5. `mycli databases get example`

The agent moves from intent to discovery, inspection, execution, and verification without needing a large command reference in its initial context.

### Search design

A command search feature should:

- Accept natural-language task descriptions.
- Search command names, summaries, descriptions, and parameter metadata.
- Return a short, ranked set of candidates as JSON.
- Include exact executable command paths.
- Work locally without authentication wherever possible.
- Support synonyms, common terminology, and user-intent language.

A search result should contain enough information to identify the correct command, but not an entire reference manual.

**Separate discovery from execution.** Search and schema inspection should never perform mutations.

### Progressive disclosure

Do not expose a CLI's entire command catalogue to an agent at startup.

Use three levels of information:

- **Orientation:** A brief description of the CLI and how to discover commands.
- **Discovery:** A small set of relevant candidate operations.
- **Inspection:** Complete documentation for a selected operation.

This keeps token consumption proportional to the task instead of the total size of the CLI.

## Enforce a consistent command grammar

Agents generalize from patterns. Inconsistent naming conventions encourage incorrect command guesses.

Cloudflare identified historical inconsistencies such as using `info`, `get`, and `describe` for conceptually similar operations.

### Recommended grammar

For resource-oriented interfaces, standardize on:

`mycli <resource> [subresource] <operation> [arguments] [options]`

For example:

- `mycli projects list`
- `mycli projects get example`
- `mycli projects create --name example`
- `mycli projects update example --name renamed`
- `mycli projects delete example`

Use a small vocabulary of operations:

| Operation | Meaning |
|---|---|
| `list` | Enumerate a collection |
| `get` | Retrieve one resource |
| `create` | Create a resource |
| `update` | Modify an existing resource |
| `delete` | Remove a resource |
| `run` | Execute a process or task |
| `status` | Retrieve operational state |
| `logs` | Retrieve logs |
| `inspect` | Retrieve detailed diagnostic information |

Other verbs are appropriate when they describe genuinely distinct domain operations.

### Naming rules

- Prefer kebab-case for command names and flags.
- Use consistent singular or plural conventions for resource groups.
- Keep operation names stable.
- Use predictable positional arguments for resource identifiers.
- Standardize common option names.
- Avoid multiple names for the same concept unless aliases are necessary for compatibility.

Enforce these rules through schema validation, generation, and automated tests rather than documentation alone.

**Consistency is more valuable than clever command names.**

## Make JSON the primary output contract

Cloudflare observed that agents frequently added `--json` and then used `jq` to extract the fields they needed.

An agent-first CLI should eliminate that unnecessary negotiation.

### Recommendations

**Default to structured output.**

For commands returning structured data, emit JSON without requiring an additional flag.

Reserve human-formatted tables and rich terminal output for an explicitly requested presentation mode.

Consider:

- JSON by default.
- Compact JSON for machine-oriented execution.
- Readable, indented JSON for interactive terminals.
- Explicit `--output table` or `--output text` when those formats add value.
- Optional NDJSON for streaming collections and event streams.

Keep the format contract stable regardless of whether terminal colors or other presentation features are enabled.

### Respect standard streams

**Standard output (`stdout`)** should contain the requested result and nothing else.

**Standard error (`stderr`)** should contain diagnostics, progress messages, warnings, and errors.

Do not mix progress spinners, authentication messages, terminal escape sequences, or decorative text into machine-readable results.

Binary-producing commands should emit raw binary data only under an explicitly documented contract or write it to a specified file.

### Return useful, structured results

For resource mutations, prefer a machine-readable receipt containing:

- Operation status.
- Resource identifier.
- Target environment or account.
- Relevant output fields.
- Job or transaction identifier, where applicable.

For collections, use a predictable schema and include pagination information when results may be incomplete.

For example, a paginated response might contain `items`, `nextCursor`, and `hasMore`.

Never silently truncate a collection in a way that causes an agent to mistake the first page for the complete result.

### Optimize for selective retrieval

Support efficient ways to reduce unnecessary data:

- Server-side filtering and search.
- Pagination and bounded limits.
- Field projection where practical.
- Output redirection and `jq` compatibility.
- Streaming for large or long-running results.

The objective is to return the information necessary for the task without filling the agent's context with irrelevant data.

## Make errors actionable and machine-readable

A nonzero exit code tells an agent something failed. It rarely tells the agent enough to recover correctly.

Design errors as part of the CLI's public contract.

### Error requirements

Every failure should have:

- A stable machine-readable error code.
- A concise explanation.
- Relevant details identifying the invalid input or failed operation.
- A clear distinction between retryable and permanent failures.
- An appropriate process exit code.
- Suggested corrective actions when these can be determined reliably.

For example:

`INVALID_ARGUMENT`, `UNAUTHENTICATED`, `PERMISSION_DENIED`, `NOT_FOUND`, `CONFLICT`, `RATE_LIMITED`, and `TIMEOUT` are more useful categories than a generic failure.

When returning structured errors, define a consistent JSON error schema, including fields such as `code`, `message`, `retryable`, and `details`.

Diagnostics should go to standard error, leaving standard output suitable for parsing.

### Important invariants

- Successful execution exits with status zero.
- Failed execution exits nonzero.
- A cancelled or refused noninteractive operation must not appear successful.
- A command must not claim success before the operation has actually completed.
- An accepted asynchronous operation must be distinguishable from a completed operation.

These guarantees allow agents to use process status and structured results without interpreting ambiguous prose.

## Make operations previewable, safe, and verifiable

An autonomous agent needs the ability to understand an operation's consequences before performing it.

### Dry-run support

Provide a consistent `--dry-run` option for mutating commands where feasible.

A dry run should:

- Validate command arguments.
- Resolve the target resource or environment.
- Describe the proposed changes.
- Identify potentially destructive effects.
- Perform no mutations.
- Avoid acquiring credentials when they are unnecessary.

Distinguish between a command that merely previews an API request and a command that computes an actual server-side change plan.

If the tool cannot determine every side effect, say so explicitly.

### Confirmation semantics

Destructive and high-risk operations should require explicit authorization.

Use a consistent mechanism such as `--force` or a dedicated confirmation flag, with documented behavior.

Do not automatically convert a missing interactive confirmation into approval simply because a command is being run by an agent.

For especially sensitive operations, support confirmation of the exact resource identifier or an approved plan identifier.

A flag used to bypass confirmation should not silently change the underlying operation's semantics.

### Verification

Wherever possible, support an agent's ability to check the result independently:

- Read back created or modified resources.
- Return stable resource identifiers.
- Expose operation and job status.
- Support idempotency keys for retryable mutations.
- Provide revision, ETag, or other concurrency checks when applicable.
- Report partial completion explicitly.

**The ability to issue a command is not the same as the ability to establish its success.**

## Make execution context explicit

An agent must know exactly where an operation will take effect.

This is especially important for tools that can operate across local development environments, remote services, multiple accounts, or production environments.

### Required context

Expose relevant context such as:

- Current project and workspace.
- Selected account or organization.
- Environment: development, staging, or production.
- Local versus remote execution target.
- Region or cluster.
- Active authentication profile.

Provide a way to inspect the context before executing commands, such as `mycli context show`.

### Safe target selection

- Standardize targeting options across commands.
- Make defaults explicit and documented.
- Surface the effective target in mutation results.
- Never silently fall back from local execution to production.
- Fail clearly when an operation does not support the requested target.
- Make switching accounts or environments deliberate.

Where local simulation is supported, use the same command and data semantics as the remote implementation whenever possible.

This helps agents reason about behavior consistently across testing and production.

## Treat configuration as an agent-facing API

Cloudflare found that TypeScript configuration improved agent performance because models could use type information, autocomplete, and language-server diagnostics.

The broader principle is that **configuration should be statically understandable and programmatically verifiable**.

### Recommendations

Use configuration formats with:

- Explicit, discoverable schemas.
- Static type checking or schema validation.
- IDE and LSP support.
- Clear error diagnostics.
- Predictable environment overrides.
- Programmatic composition where appropriate.

TypeScript configuration is particularly useful when a system benefits from conditional values, reusable components, computed environments, and complex relationships.

JSONC with JSON Schema may be preferable for tools that require portable, declarative configuration without executing code.

Do not assume TypeScript is universally superior. Choose a format based on the operational requirements of the application.

### Additional requirements

Provide commands to:

- Validate configuration without applying it.
- Inspect the resolved effective configuration.
- Preview configuration-driven changes.
- Distinguish missing values from explicit nulls and defaults.
- Identify deprecated or unsupported fields.

Avoid embedding secrets directly in configuration files.

If configuration is executable code, account for side effects, untrusted dependencies, and reproducibility during evaluation.

## Design for composability and automation

Agents frequently perform multi-command workflows rather than isolated operations.

The CLI should support ordinary shell composition without requiring custom integrations.

### Input conventions

Prefer:

- Named flags for simple scalar inputs.
- Structured JSON input for complex objects.
- `@file.json` conventions for large payloads.
- Standard input where appropriate.
- Explicit `--body` or equivalent for complete request bodies.
- Consistent support for arrays, enums, and nested objects.

Avoid requiring an agent to construct extremely long shell commands containing deeply escaped JSON.

### Noninteractive execution

Every operation intended for automation should have a documented noninteractive path.

When required input is absent:

- Prompt interactively when appropriate for humans.
- Return a clear validation error for unattended agents.
- List missing fields and accepted values.
- Never hang indefinitely waiting for input.

Treat structured validation errors as an opportunity for the agent to repair a command.

### Long-running operations

Support:

- Stable operation identifiers.
- Machine-readable status.
- Explicit timeouts.
- Cancellation.
- Progress reporting without corrupting standard output.
- Optional waiting until completion.
- Retrieval of logs and results after execution.

This allows agents to manage operations that outlive an individual shell invocation.

## Make documentation available on demand

Documentation should be retrievable rather than assumed to exist in the model's training data.

Cloudflare's approach demonstrates a useful separation between command discovery and documentation search.

### Recommendations

Maintain distinct interfaces for:

- **Command search:** Which operation should I execute?
- **Command schema:** How exactly do I invoke it?
- **Documentation search:** What concepts, prerequisites, and examples do I need?
- **Version inspection:** Which behavior is supported by the installed CLI?

Prefer machine-readable search results containing concise descriptions and documentation locations.

Make full documentation available as Markdown or another easily parsed text format.

Avoid requiring agents to extract knowledge from interactive documentation websites.

### Agent instruction files

Provide a compact `AGENTS.md` or reusable agent skill that explains:

- When to use the CLI.
- How to search for commands.
- How to inspect command schemas.
- How to preview potentially harmful changes.
- How to interpret results.
- Important platform-specific limitations.

Keep this document small. It should direct agents toward live discovery, not duplicate the entire reference manual.

### CLI and MCP should share the same foundation

An MCP server can expose the same operations through structured tool calls, but it should not require maintaining a second independent definition of the product's capabilities.

Generate MCP tools, command metadata, and documentation from shared schemas wherever possible.

Use whichever interface is appropriate for the execution environment without introducing inconsistent semantics.

## Design authentication for unattended agents

Authentication should be explicit, scoped, and secure.

### Recommendations

Support:

- Interactive sign-in for human users.
- Noninteractive authentication with appropriately scoped tokens or identities.
- Named profiles for different accounts and environments.
- Least-privilege permissions.
- Credential inspection and revocation.
- Unambiguous authentication errors.

Never require unattended agents to complete browser-based login flows.

Do not print secrets, credentials, or sensitive authorization headers into ordinary output, errors, dry runs, or logs.

Avoid putting sensitive credentials directly in command-line arguments because process arguments can be exposed through logs and operating-system facilities.

Use secure credential stores, protected files, environment injection, or other suitable secret-delivery mechanisms.

Command discovery and local schema inspection should work without credentials whenever practical.

## Establish a complete agent interaction contract

The CLI should expose enough information for an agent to follow a repeatable sequence:

**Discover → Inspect → Validate → Preview → Execute → Verify**

Each step has a distinct responsibility.

| Phase | CLI responsibility |
|---|---|
| Discover | Locate relevant operations from task intent |
| Inspect | Describe arguments, constraints, outputs, and side effects |
| Validate | Reject malformed or unsupported inputs |
| Preview | Show the target and intended effects |
| Execute | Perform the authorized operation |
| Verify | Expose the resulting state or completion status |

Not every read-only command needs every phase. The important property is that the workflow is available whenever the operation requires it.

A CLI implementing this contract should allow an unfamiliar agent to accomplish a meaningful task without consulting external documentation or guessing at command syntax.

## Measure agent usability directly

Traditional CLI quality measures such as command count and documentation coverage are insufficient.

Test the interface using agents that begin with no specialized knowledge of the CLI.

### Recommended evaluation metrics

| Metric | What it measures |
|---|---|
| Task completion rate | Whether the intended result was achieved |
| Command discovery accuracy | Whether search identifies the correct operation |
| Invalid command rate | How often agents invoke nonexistent or malformed commands |
| Tool-call count | Commands required per successful task |
| Context consumption | Tokens spent on discovery, documentation, and output |
| Recovery rate | Whether agents recover from predictable errors |
| Verification rate | Whether agents confirm results rather than assume success |
| Unsafe-operation rate | Whether agents cause unintended or unauthorized changes |
| Cross-command consistency | Whether learned conventions generalize |
| Version resilience | Whether agents use current schemas rather than outdated knowledge |

### Evaluation scenarios

Test unfamiliar agents against representative tasks, including:

1. Discovering and executing an unknown command.
2. Creating, modifying, and deleting resources.
3. Handling missing required arguments.
4. Selecting between local and remote environments.
5. Processing large or paginated results.
6. Recovering from authentication and permission failures.
7. Handling rate limits and transient errors.
8. Detecting unsuccessful or partially completed operations.
9. Performing a multistep workflow without previously learned command syntax.
10. Operating safely when documentation, defaults, or versions change.

Include adversarial cases involving ambiguous resource names, misleading results, and attempts to bypass confirmation requirements.

Measure actual end-state correctness rather than treating a zero exit code as sufficient evidence.

## Final recommendations

Cloudflare's experience supports a shift in how command-line interfaces should be designed.

The traditional assumption is that a developer learns the CLI and then invokes commands.

An agent-first CLI should instead allow an agent to learn the relevant interface dynamically while working.

The most important architectural decisions are therefore:

1. **Make the command schema authoritative**, generating interfaces from it instead of maintaining separate interpretations.
2. **Provide native command discovery**, enabling agents to work without prior training knowledge.
3. **Use structured input and output contracts**, making the CLI predictable and composable.
4. **Enforce consistency throughout the command tree**, allowing agents to generalize reliably.
5. **Make side effects explicit and results verifiable**, especially for autonomous operations.
6. **Use progressive disclosure**, keeping the agent's context focused on the task.
7. **Evaluate the complete agent workflow**, not just the correctness of individual commands.

The goal is not to make agents better at operating traditional CLIs. It is to design CLIs whose capabilities can be discovered, understood, and used reliably by agents.

**A good agent-first CLI should be usable by a capable agent with no prior knowledge of the tool, no special prompt engineering, and no manual intervention for ordinary authorized operations.**

## Sources and further reading

- [Cloudflare — Introducing cf: the agentic CLI for the entire Cloudflare API](https://blog.cloudflare.com/cloudflare-cf-cli-launch/), September 28, 2026.
- [Cloudflare — Building a CLI for all of Cloudflare](https://blog.cloudflare.com/cf-cli-local-explorer), April 13, 2026.
- [Cloudflare CLI — Using cf with coding agents](https://developers.cloudflare.com/cf/agents/).
- [Cloudflare CLI source repository](https://github.com/cloudflare/cf).
- [Cloudflare's official CLI agent skill](https://github.com/cloudflare/skills/blob/main/skills/cf-cli/SKILL.md).
- [Cloudflare CLI changelog](https://github.com/cloudflare/cf/blob/main/packages/cli/CHANGELOG.md).

*The Cloudflare references document the observed usage patterns and specific implementation choices. The broader interface contracts, testing criteria, and security recommendations in this guide are proposed engineering practices, not claims that Cloudflare implements every feature described.*
