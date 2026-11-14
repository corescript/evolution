# A Vision for CoreScript

> This is a prospective vision document. Nothing described here is implemented yet. It records the goals of the language and the basic approach to reaching them, in enough detail to argue with. Concrete designs will be proposed and reviewed separately, and that review may reject or revise what is sketched here. All syntax and every command shown are illustrative.

## Introduction

A product that ships as a server, a web app, an iOS app, and an Android app has four implementations of much of its logic and four sets of tests, and all of them must agree. Making the server authoritative removes some of the duplication, and generating types removes a little more. Validation, state machines, formatting, navigation, caching, and offline behavior are still written once per platform. The copies drift, and they drift faster now that products change every week.

Agents write most of that code. The cost of writing it has collapsed, and the cost of knowing it is right has not. Humans review evidence more often than they review code: preview builds, screenshots, test results, changed contracts. With four implementations, the question a reviewer cares about, whether the code meets the requirements, is tangled up with a second one, whether the implementations agree. The second question is mechanical. A shared language answers it once, as a matter of compiler conformance, and leaves human and agent judgment for the first.

Sharing has a limit, and we draw it on purpose. A product should be built with the toolkit native to each platform: the DOM on the web, SwiftUI and UIKit on iOS, AppKit on macOS, Compose and views on Android, WinUI on Windows. Different presentations are the cost of doing multiplatform apps well, and this vision does not try to remove that cost. Beneath presentation sits a large body of logic with no reason to differ between platforms: business rules, the state machine behind each screen, service contracts, and the interface logic that never touches a platform convention. That body is the core.

CoreScript is a programming language for the core, together with the tools an agent needs to work in it. The language is a dialect of TypeScript. It keeps TypeScript's syntax and type system, removes the dynamic behavior that cannot mean the same thing on every platform, and adds value types, checked casts, effect clauses, and actors. Its tools assume that the author is an agent and the reviewer is a person looking at evidence.

Eight things define CoreScript, and the main sections of this document take them in order:

- A [content-addressable codebase](#a-content-addressable-codebase). Code is stored as definitions keyed by hash, and text is a view.
- [Effect metadata](#effect-metadata). Every definition carries what it can throw, which capabilities it uses, and where it must run.
- [Distributed package registries](#distributed-package-registries). Releases are signed records on the AT Protocol, and anyone can run an index.
- [Agent-first source control](#agent-first-source-control). History, merging, and publication work on definitions and are built for many agents at once.
- An [agent-first terminal interface](#an-agent-first-terminal-interface). One schema-driven command line that an agent can discover, preview, and verify with no prior training.
- [Agent-first language intelligence](#agent-first-language-intelligence). The compiler answers questions about meaning as structured data.
- [Multi-target compilation](#multi-target-compilation). JavaScript for the web, native code for Apple platforms and Windows, class files for Android.
- [Multi-toolkit user interfaces](#multi-toolkit-user-interfaces). One core under an interface written for each platform's toolkit.

The closest existing answer is Kotlin Multiplatform. The code it generates for JavaScript and Wasm is less than optimal, and Kotlin carries too much implicit meaning and too little verification for code that agents write and people skim. Frameworks that share the interface along with the logic give up the native presentation we want to keep. CoreScript's bet is narrower than either: one implementation of the core, native presentation above it, and evidence attached to exact code.

## Principles

Eight principles govern the design. Later sections refer to them by number.

1. **Agents are the primary users.** The language, documentation, and tools should be designed for agents to discover operations, make changes, observe behavior, and verify results. Humans set intent and judge outcomes through running software and evidence. A feature is incomplete until an agent can exercise and inspect it.
2. **Make trust mechanical.** Move every obligation the system can reliably check into the compiler or verification tooling. Distinguish static guarantees from test evidence and human judgment. Every guarantee has an explicit boundary, and every verification result belongs to exact inputs. Human attention should concentrate on intent and the obligations that remain.
3. **Make contracts explicit.** Failure, suspension, and execution context belong in declarations. Inputs that affect application behavior enter through declared state or capabilities. An agent should be able to use an abstraction correctly from its contract. APIs should be self-documenting. Any and all conveniences must preserve the ability to reason locally.
4. **Specify once.** Express schemas, state transitions, service contracts, and other mechanically checkable specifications as authoritative declarations. Derive the artifacts that platforms need from them. Give portable code one defined meaning, and hold every backend to it. Whenever two representations must agree, make their agreement the system's responsibility.
5. **Be native on every platform.** Use the host's controls, libraries, and runtime primitives. Preserve shared behavioral contracts while allowing platform-specific presentation and interaction. Prefer an explicit platform implementation when a universal abstraction would sacrifice fidelity. On the web, that includes respecting download size and using JavaScript's existing primitives.
6. **Make meaning the unit of work.** Organize tooling around definitions, dependencies, and intended changes. Names and source files are ways to navigate and edit that structure. Review should expose changed contracts and affected behavior; verification should follow actual dependencies. Formatting should never determine which changes conflict.
7. **Preserve exploration; verify publication.** Agents must be able to save, share, and recover unfinished work, including invalid drafts and unresolved alternatives. Keep those operations separate from making a revision authoritative. Publication requires the evidence and approval for the exact candidate being published. Parallel work must preserve other agents' changes and the history needed to understand them.
8. **Grow through libraries.** Keep the language's semantic core small. Build domain vocabularies through ordinary types and userland composition before introducing compiler features. Provide extensive, canonical, well-documented libraries so agents have a consistent target. Specialized abstractions must remain subject to the same verification rules as ordinary code.

Principle 4 asks for one meaning everywhere, and principle 5 asks for the host's own primitives on each platform. That tension is the project. We resolve it by giving the core one meaning, built on JavaScript's semantics, and by ending shared code at a seam that each platform consumes in its own language.

## A content-addressable codebase

CoreScript has no source files. The codebase stores one tree per definition (each function, type, and constant), addressed by the hash of its content. A name is a path that points at a hash, and a namespace is a prefix of such paths. References between definitions are by hash. The model is Unison's, and [tcm](https://tcm.codes/) has shown that it works for TypeScript.

Several properties follow directly from that storage model, and most of the rest of this document depends on them.

- A definition means the same thing until someone deliberately moves a name. Nothing changes underneath a caller.
- A rename moves a label. No code is edited, and nothing that used the old name breaks.
- The codebase holds the whole dependency graph, so it knows exactly what a change reaches, and a build contains only what it reaches.
- Two versions of a library can sit side by side, because they are different hashes.
- Facts about a definition attach to its hash and cannot go stale: its effects, its test results, the expansion of its decorators, the evidence gathered about it.

Maintaining a code database by hand is tedious for a person. For an agent it is the natural interface, with all of the benefit and none of the drawbacks.

### Views

Syntax is a parser and a printer over the stored trees. That has three consequences. Formatting cannot conflict, because it is not stored. A reading aid does not have to be a writing burden: a view can print inferred effect clauses beside each signature, argument labels at call sites, the expansion of a decorator, or a reducer as a transition table, and nobody has to type any of it. And the surface syntax can be varied or replaced without touching stored code, which is what makes the benchmark described under [Verification](#verification) cheap to run.

There is a lossless canonical text form. Review and editing need one, and it makes a git mirror of a codebase cheap. When files have to be materialized, they are a rendering of the store and never its source of truth.

### Changes that carry through

References by hash change what a conflicting edit looks like. Suppose one agent changes a function's signature while another adds a caller. Git merges the two cleanly, and the build breaks. Here the new caller stays valid against the old definition, and it appears on a list of things to update when the name moves. Dependents that still type-check follow a change automatically. Those that do not keep their old version and go on the list, so nothing is ever half-updated. The mechanism is tcm's, and CoreScript leans on it: a function that starts throwing puts every caller that lacks a `try` on that list.

### Evidence by hash

Principle 2 asks that every verification result belong to exact inputs, and hashes are how the system knows what the inputs were. A test result is recorded against the hashes it exercised, and when the test has no effects, it is a pure function of those hashes and can be cached indefinitely and shared between machines. Decorator expansions and compile-time results are cached the same way. We intend evidence in general to be recorded as a verdict about a specification hash, an implementation hash, and a checker hash, so that deciding whether a candidate is fit to publish becomes a query.

## Effect metadata

Every definition in the codebase carries three facts about its behavior, computed by the compiler and stored against its hash: how it can fail, what it needs from the world, and where it must run. Principle 3 says failure, suspension, and execution context belong in declarations. JavaScript already marks suspension with `async` and `await`, and effect metadata supplies the rest.

The metadata is inferred for every definition, so it costs nothing to write. On exports it is written as clauses that the compiler requires and checks, so a public contract cannot change by accident.

```ts
export async function loadCart(id: CartId): Promise<Cart>
  throws(NotFound | NetworkError)
  uses(network)
{
  const response = try await fetch(cartUrl(id))
  return try parseCart(await response.text())
}
```

| Clause | Says | Propagates through | Discharged by |
|---|---|---|---|
| `throws(E)` | the call may fail with `E` | a call marked `try` | `catch` |
| `uses(C)` | the call needs capability `C` | any call | a scoped handler |
| `isolated(A)` | the call must run in actor `A`'s domain | a synchronous call | `await` from another domain |

### Failure

A call that can throw must be marked, with three forms: `try f()` propagates the error, `try? f()` evaluates to `T | undefined`, and `try! f()` traps. We considered rendering the marker in views without requiring it in what agents write, and chose to require it. A stored codebase should have one spelling per meaning. A required marker also makes a contract change loud: when a function starts throwing, its callers stop compiling and land on the list of things to update, where a silently inferred marker would let them follow along unnoticed. With every throwing call marked and every `throws` clause known, `catch (e)` binds a real type, the union of what its `try` block can throw, where TypeScript gives `unknown`.

### Capabilities

`uses` names the capabilities a function needs: the network, the clock, randomness, the file system, a database. Code still calls `fetch` and `Date.now()` directly. The clause records that it did. Tests replace a capability with a scoped handler, which every runtime has to carry across `await`. A capability can have several implementations, so `uses(d1)` can be Cloudflare's database on a server, SQLite on a phone, and a fake in a test, as long as the capability's contract promises only what all of them provide.

### Isolation

`isolated` names the actor whose state a function touches. It is described with the rest of the concurrency model under [Actors and isolation](#actors-and-isolation).

### What the metadata is for

In this design, effect metadata began as bookkeeping that the tools would keep for each definition, with no syntax at all. The clauses came later. The bookkeeping is still the point, and a good deal of the system reads it.

- Availability and placement. An export exists on a target when every capability in its `uses` set is implemented there. Code that reaches for the file system is a compile error in a browser build and compiles everywhere else, with no separate availability annotation to maintain.
- Sound caching. A test whose `uses` set is empty is deterministic, so its cached result can be trusted.
- Compile-time evaluation. An expression with no effects is safe to run during a build.
- Review. A change is summarized in terms of contracts, such as "this change adds `network` to `checkout`".
- Policy. Tooling can enforce rules per namespace, for instance that nothing under a domain layer uses the network.
- Supply chain. A library's effects are visible before anyone depends on it, and a release that starts using the network shows up as a changed contract.
- Architecture. A reducer declared with no effects is checked to have none, which turns "no clock, no random, no I/O" from a convention into a compile error.

Three alternatives stay out. Passing capabilities as parameters and returning errors as values would each rewrite every function body or every call, for information the compiler can carry in a clause. Resumable effect handlers in the style of Unison need continuations, which none of our hosts expose.

## Distributed package registries

CoreScript libraries are published without a central registry. A release is announced as a signed record on the [AT Protocol](https://atproto.com/), the protocol behind Bluesky, and a registry is an index that anyone can run. tcm distributes libraries this way, and Zena has [designed its package manager](https://github.com/elematic/zena/blob/main/docs/design/package-manager.md) around the same protocol. We intend to follow them closely.

The protocol supplies pieces that a package ecosystem usually has to build and then govern. An account is named by a DID, which stays the same when the account changes its host or its handle. An account's records live in a repository whose commits are signed with the account's key, so any single record can be fetched together with a proof that it belongs there. A handle is a domain name the account proves it controls. Indexes and moderation are separate services that read the same public data.

That maps onto a content-addressed codebase with little left over.

- A release is a map from names to definition hashes, with the contracts of its exports. It is a record in its publisher's repository.
- Identity is a DID, and a library is found through a handle under a domain its maintainers control. No global namespace of short names exists to squat on.
- A registry finds release records on the network, checks each against its publisher's signed repository, and mirrors the code, re-hashing every definition as it arrives. The hash is the identity, so a mirror or a cache needs no trust.
- A dependency is pinned by hash from the moment it is added.

Content addressing and effect metadata add things a registry of tarballs cannot offer. A registry can show what each version changed in terms of definitions, and can check a version number against that difference. It can show what a library needs before anyone installs it: which capabilities it uses, which targets it is therefore available on, and whether a patch release has started reaching for the network. Test results recorded against hashes can travel with a release.

Trust is layered on the same records. Maintainers can attest to a release from their own accounts, a package can require several attestations, and independent labelers can mark a release as reviewed, deprecated, or malicious, with each client choosing whose labels to follow. Zena's design works these mechanisms out in detail, including a public log that pins the first content seen for each version.

npm remains the source of foreign code, handled as described under [Packages](#packages). The AT Protocol registry is for libraries written in CoreScript. None of this is designed yet, and one early choice is between Zena's model, where a package is an account of its own, and tcm's, where a release is announced from its author's account.

## Agent-first source control

Git stores files and lines, expects a few people who take turns, and loses work when a step is forgotten. A codebase of definitions edited by many agents at once needs different properties, and most of them fall out of the storage model.

History is a history of names. Each revision is a map from names to hashes, and the difference between two revisions is stated in terms of meaning: this definition was renamed, this one changed, these were re-hashed only because something they use changed. A merge is three-way by name, so a conflict means both sides changed the same definition. Formatting cannot conflict, and neither can two edits that happen to touch neighboring lines.

Principle 7 sets the rest of the requirements, and systems already built for agents show how to meet them. We expect to borrow from [Veil](https://github.com/RKKatakam/veil), from [jj](https://github.com/jj-vcs/jj), from tcm, and from Anthropic's account of [Scion](https://www.anthropic.com/research/redesigning-source-control).

- Nothing is lost by forgetting a step. There is no staging area. Every operation is recorded, and undo adds a new entry and rewrites nothing.
- Unfinished work can be stored and shared. A draft may be text or a tree with holes, may fail to compile, and may hold alternatives nobody has chosen between. Saving a draft never makes it authoritative. A store that accepts only typed trees, as Unison's does, cannot hold one.
- A merge always finishes. A conflict is stored as data in the change, to be resolved later by the same agent or another.
- Each agent works in its own branch or workspace, starting from the newest revision that passed verification. A server lands finished changes on the main line, so nobody pulls, rebases, and retries.
- Dead ends are recorded with their reasons, so the next agent does not repeat them.
- Every change carries signed attribution: which agent made it, for which task, and on whose behalf.
- Publication is gated. Making a revision authoritative requires the evidence and the approval for that exact candidate, and since evidence is recorded against hashes, the gate is a query.

Review changes shape as well. A reviewer sees what a change reaches, which contracts it alters, and which tests cover it, and reads code only where that summary raises a question. Verification follows the same graph: the tests that reach a change are the ones that run, and their results travel with the change to every other machine.

Source control here is a service that answers questions, such as what changed, who changed it, and what depends on it, with no checkout needed to ask. Git cannot operate on this model, and we do not want lossy conversions to files in order to use it. The lossless text form makes a read-only git mirror cheap, for hosting and for tools that only read.

The commitments above are firm. The mechanisms behind them are the least designed part of this vision.

## An agent-first terminal interface

With no files, the command line is the way into the codebase. Adding a definition, rendering a view, running the tests a change reaches, building for a target, and publishing a release are all commands. The design follows the guide in [docs/agent-first-clis.md](docs/agent-first-clis.md), whose test is that a capable agent with no prior knowledge of the tool can use it with no special prompting.

- One schema describes every command: its arguments, types, and defaults, its output, whether it reads or changes state, and the errors it can return. The argument parser, the help text, the machine-readable metadata, the MCP tools, and the documentation are generated from it, so they cannot disagree.
- The interface is discoverable from inside. An agent searches for a command by describing its intent, inspects the schema of the one it picks, and reads documentation on demand, including documentation for the language itself. Discovery never changes anything.
- Commands share one grammar, a resource followed by an operation, with a small fixed vocabulary of operations. Agents generalize from patterns, so consistency is worth more than clever names.
- Output is structured by default. Results go to standard output as JSON and diagnostics go to standard error. A collection says when there is more of it.
- Errors are part of the contract. Each has a stable code, says whether retrying can help, and suggests a correction when one is known. Compiler diagnostics follow the same rules.
- Anything that changes state can be previewed with a dry run, and its result can be read back afterward. Destructive operations need explicit authorization, and running without a terminal never counts as consent.
- Context is explicit: which codebase, which branch, which target, which registry.

```sh
core search "rename a definition"
core schema definitions move
core definitions move shop.total shop.cartTotal --dry-run
core definitions move shop.total shop.cartTotal
core definitions get shop.cartTotal
```

That sequence is the contract in miniature: discover, inspect, preview, execute, verify. The same operations are available over MCP, generated from the same schema.

We will measure this interface the way we measure the language, with agents that have never seen it. The metrics are task completion, invalid command rate, calls per task, context consumed, and how often an agent verifies a result where it could have assumed one.

## Agent-first language intelligence

Language servers were designed for a person at a cursor: hover here, complete at this position, jump from this line. An agent asks different questions. It wants answers about the whole codebase, as data, sized to fit a context window, and it wants to know what an edit did before it moves on. The codebase already holds what those answers need, because every reference is exact and every definition's contract is computed.

The tooling answers questions like these directly:

- What is this definition's contract? The answer is the effective signature, with inferred effects and decorator expansions included.
- What does it use, and what uses it? Dependencies and dependents come from the graph, with no text search and no false matches.
- What would this change reach? That covers the definitions that follow, the ones that stop compiling, the tests that cover them, and the exports whose contracts move.
- Where is the code that does this? Search works by name, by text, by type (matched up to renaming and argument order, as tcm does), and by effect, such as everything that uses the network.
- Where does this run, and on which targets is it available?
- What evidence exists for this hash?

Edits are operations on meaning. An agent replaces a definition and gets back what followed, what landed on the update list, and which tests ran. A rename moves a label. A larger refactor is a tree transformation that the tool applies and checks as one step.

Diagnostics are written for a repair loop. Each has a stable code and names its fix, in the way that a rejected downcast points at `as?` and `as!`. A compile error is the cheapest feedback an agent gets, which is why the language prefers an error to a silent difference everywhere, and the tooling's job is to make each error sufficient to act on.

Views control how much an agent reads. A signature-only view of a namespace, a contract view of one definition, and a transition table for a reducer are renderings of the same trees, and an agent asks for the smallest one that answers its question.

Observation completes the loop. Principle 1 says a feature is incomplete until an agent can exercise and inspect it, so a running program can be driven and inspected through the same interface: dispatch an action to a component and read its state, with no user interface attached.

All of this is served from the same schema as the terminal interface, and over MCP. A conventional language server remains available for people in editors, since the checker we fork includes one.

## Multi-target compilation

One codebase compiles to every platform a product ships on, and each target gets output native to it. CoreScript does not ship a JavaScript engine inside a native app. The goal is a native language that is smaller and faster than a JavaScript runtime. It also does not emit Swift or Kotlin source for shared code, because output that must type-check in two more languages would tie CoreScript's semantics to their limits.

| Target | Output | Memory |
|---|---|---|
| Web | JavaScript modules. No Wasm, and no shipped runtime beyond inserted checks and value equality. | The engine's garbage collector |
| Server | JavaScript on an existing engine, where any npm package also runs. A native backend is not designed. | The engine's garbage collector |
| Apple platforms | Native code behind a C ABI with a generated Swift overlay. Emitting Swift's ABI directly can follow and would remove the overlay's marshalling. | Reference counting |
| Android | JVM class files that D8 and R8 turn into DEX, with Kotlin metadata so that Kotlin callers see real `suspend` functions and sealed types. | The host's garbage collector |
| Windows | Native code with a COM-compatible surface. Not designed. | Reference counting |

The same rules apply on every target. A program that compiles for one target means the same thing on all of them, and the web build carries the same inserted checks as the native ones. The memory rule is that a process never holds two garbage collectors: natively the runtime counts references, and on Android it uses the collector that is already there. Finalization timing is unobservable everywhere. Whether the reference-counted runtime needs a cycle collector is open.

### The seam

The seam is the boundary between shared code and presentation. On each platform it is an exported API typed in the host's language, so type-checking against Swift or Kotlin applies to the exported surface and never to function bodies. What crosses it is data, state going out and actions coming in, which means a session can be traced and replayed.

| CoreScript | Swift | Kotlin |
|---|---|---|
| struct, record, tuple | struct | data class |
| enum | enum | enum class |
| enum with payloads | enum with associated values | sealed interface |
| `int` | `Int` | `Long` |
| async function | async function | `suspend` function |
| parameter names | argument labels | named parameters |

The host compiler does some of the checking. A SwiftUI view that switches over an exported state enum and ignores a case fails to compile, in Swift, with no CoreScript tooling involved. We expect an export isolated to the main actor to appear that way to Swift, and to reach Kotlin through a generated wrapper that hops to the main dispatcher, since Kotlin has no static isolation.

### Calling the platform

The codebase is its own declaration format. A library release is a set of signatures and hashes, and nothing separate has to be generated or kept in sync with it. Foreign APIs enter through importers that translate a platform's declarations into CoreScript signatures:

- TypeScript declaration files, for npm packages and the web platform.
- C and Objective-C headers, read through Clang, which covers UIKit, AppKit, and Foundation.
- JVM class files together with Kotlin's metadata.
- Windows metadata files, which describe WinRT and, through Microsoft's win32metadata project, Win32 and COM.

Swift-only APIs are the exception. Calling them needs Swift's generics and witness-table ABI, so the route to them is a small shim written in Swift, and we do not plan to import Swift interface files.

Foreign formats cannot state everything a CoreScript signature carries: `throws`, `uses`, isolation, `int`, records, and nullability the original got wrong. Overlays supply the difference. An overlay is a sidecar keyed to foreign symbols, so it survives package updates where a forked declaration file would conflict; Apple's API notes are the precedent. For npm packages, whose source is JavaScript or TypeScript, the compiler reads the source and infers effects as it does for code written in CoreScript, and overlays cover the remainder. Platform capabilities are declared by hand at the bottom: the binding written over `NSURLSession` is where `uses(network)` is defined.

### Packages

An npm package is foreign code, in the way UIKit is. The dialect governs what you author and says nothing about a dependency's internals. On the web and on a server, any package runs on the engine as it always has. Its declaration file is a claim nobody verified, so values arriving from it are either checked at the boundary or accepted with the trust recorded.

Inside the shared core, on native targets, a package has to be compiled. One that conforms to the dialect compiles as is. For one that does not, the usual answer is to port the slice the product uses, with the package's own tests as the oracle; the [calibration brief](docs/agentic-dev-calibration-brief.md) documents how cheap that has become. A package's test suite passing on each backend is the evidence that it works there.

### One standard library over a small native floor

CoreScript offers the web-standard APIs on every target: `fetch`, streams, `URL`, and the rest. They are the canonical library. Node's APIs are provided as a compatibility layer for npm packages, since they are defined by one implementation's behavior. A native app needs these APIs and has no use for a JavaScript engine to run them, so we reimplement them and embed nothing.

The API layers are written in CoreScript over a small native floor per platform: the host's HTTP stack or sockets, files, the clock, randomness, and cryptographic primitives. The layers compile once for every backend. Web Platform Tests and Node's own test suite verify them once, and each platform has only its floor to verify. Availability differs between targets and meaning does not.

Service-shaped platform APIs fit the same model. A key-value store or a SQL database in the style of Cloudflare's Workers bindings is a capability with several implementations: the hosted service on a server, SQLite on a device, and a fake in a test.

### Values in memory

A record's layout is known from its type, so native backends keep records on the stack, in registers, or inline in a parent record, class field, or array, with no header and no reference count. Assigning to a field through a `let` binding compiles to a store into one slot. Reference counts remain only on the heap storage a record can point to: strings, variable-length tuples, recursive records, and records large enough that the compiler chooses to box them. That storage is immutable and may cross between actors, so we expect its counts to be atomic, while class instances stay inside one actor and can use cheaper non-atomic counts. On the web, records are ordinary garbage-collected objects.

## Multi-toolkit user interfaces

Principle 5 rules out a shared interface toolkit. CoreScript shares everything beneath the interface and leaves the interface to each platform: the DOM on the web, SwiftUI, UIKit, and AppKit on Apple platforms, Compose and views on Android, WinUI on Windows. Two mechanisms connect the core to a toolkit. The host language can build views over the seam, and CoreScript's own markup can target a toolkit through a builder.

### One architecture under every toolkit

The language is designed against a specific shape of application, set out in [docs/ui-architecture.md](docs/ui-architecture.md). It is The Elm Architecture with its informal parts made checkable. Its one rule is that nothing is ambient: every input to behavior is a declared value or a function of state. Most of that document turns into something the compiler checks.

- State is a struct. Stored fields are fields and derived fields are getters, which cannot be assigned by construction.
- Actions are enums with payloads. The three provenances the spec names (intents from the view, completions from effects, emissions from subscriptions) are three nested enums, and a view's `dispatch` accepts only intents.
- `Update` is a function with no `uses` clause and no `throws` clause. Its body is one `match` over the state and the action, with clauses tried in written order, and the stored tree can be rendered as the spec's transition table for review.
- Effects are records. Tests compare them by value, and an effect's completions, key, and concurrency policy are static declarations.
- Lenses need no library. A state lens is a field plus the `let` update rule, and an action prism is a case pattern.
- The runtime shim that interprets effects against services is an actor.

A pure `Update` has consequences beyond tidiness. Its tests can be cached soundly, a recorded sequence of actions replays exactly, and an agent can drive a component with no interface attached by dispatching actions and inspecting state. Everything above the view has one implementation, so the spec's per-platform clauses shrink to the view, the service interpreters, and the conventions for mount and cleanup.

We considered TC39's Signals proposal as the observable primitive and prefer this architecture, whose state is a value and whose rendering is a pure function of it. The language does not force the choice, because the builder decides: one builder can subscribe holes to signals, and another can diff hole values between renders.

One change to the spec follows from this design. Subscriptions are currently `Observable<Action>[]`, which cannot be compared. They should be descriptions, like effects, so that they diff and test the same way.

### Views in the host language

On native platforms, Swift and Kotlin receive a state value and a `dispatch` function and build the view with the platform's own toolkit. SwiftUI and Compose both depend on features of their own compilers, so views for them are written in Swift and Kotlin. The host compiler then checks the view against the exported types, and a view that ignores a state does not build.

### Markup and builders

CoreScript has its own markup for the web, and for any toolkit a builder is written for. It builds on JSX's tags and changes how they compose. Inside a builder block, written `@{ }`, every statement contributes to the result. Text is an ordinary quoted string, and `if` and `for` are the language's own, with no template-only variants. Two cleanups ride along from the JSX 2.0 discussions: HTML entities are not decoded, and attributes have a shorthand.

```ts
const view = @{
  <h1>"Cart"</h1>
  if (items.length === 0) {
    <p>"Your cart is empty"</p>
  } else {
    <ul>
      for (const item of items; key item.id) {
        <li>item.name</li>
      }
    </ul>
  }
}
```

A builder block has one mode. Every `if` in it is structural, and statements that would make structure dynamic, such as `break` and reassignment, do not compile there. That property is what lets SwiftUI and Compose use plain control flow, and it is why the block carries one sigil where other designs mark each keyword. `@{ }` sits beside `#{ }` and `#[ ]` on purpose. `do { }` runs statements and yields the last value, and `@{ }` builds from all of them.

The semantics come from Swift's result builders. Markup lowers to static calls on a builder type chosen by the expected type, and builders are ordinary libraries. A DOM builder comes first. A builder for UIKit, AppKit, Android views, or WinUI becomes possible once that toolkit has been imported, and it is a library like any other. Plain JSX calls one factory per element and hands the library a fresh tree of generic nodes on every render. A builder lowering passes on what the compiler knew:

```ts
Builder.block(cartTemplate, [
  Builder.either(items.length === 0,
    () => Builder.block(emptyTemplate, []),
    () => Builder.block(listTemplate, [
      Builder.each(items, item => item.id,
        item => Builder.block(rowTemplate, [item.name])),
    ])),
])
```

Each template is a compile-time constant with a content hash, and a render is template identities plus the values in their holes. Comparing two renders compares a few values, which is the block virtual DOM technique, and the library never has to rediscover that a heading is static or that a branch has two arms. View identity falls out of the same structure: a template at a slot path is one view, the other arm of an `either` is a different one, and each row of an `each` is identified by its `key`. Because the hashes are stable across builds, hot reload and hydration can match the views that did not change.

Keys stay explicit. Records have equality and hashing from the compiler, so `key item` is always available, and it is right for items with no lifecycle, such as tags. An entity that changes over time needs a stable identifier, or editing its title would destroy and recreate its row.

## The language

The eight parts above rest on a language, and this section describes it. CoreScript's surface is TypeScript's wherever TypeScript already says what we mean. Each addition closes a gap between what a signature states and what the code behind it can do, and each lowers statically, with no runtime protocol and no reflection facility behind it. Several additions come from TC39 proposals at Stage 1 or 2. Those designs are still moving, so the syntax here is a sketch, and whatever CoreScript adopts becomes its own design from that day.

### Starting from TypeScript

We began by designing a new language and stopped. A new language has no training corpus, no tools, and no second implementation to check it against, and all of that effort would go into syntax and semantics nobody needs to be new. Agents already write TypeScript well, from a very large corpus. It has mature tooling, including dozens of Rust crates. Its web target is the language itself. JavaScript engines and test262 already define what a correct program does. Other projects have shown the remaining pieces work: [scriptc](https://github.com/vercel-labs/scriptc) and [Porffor](https://porffor.dev/) compile it ahead of time to native code, and tcm stores it as a content-addressed codebase.

TypeScript as it stands cannot be the language. Its types can lie, it has no way to state failure or capabilities, and it has no value types. So CoreScript is a dialect, neither a strict subset of TypeScript nor a superset. That is a real cost, and we judge it necessary for the best experience for both people and agents. It puts the novelty where the thesis is: a single implementation, the seam, and the tooling for evidence.

People still read this code, so it has to be easy to read and easy to catch mistakes in. We add annotations where they state a contract and avoid them where a small amount of runtime can do the job. CoreScript has no ownership or lifetime annotations; reference counting and the host's garbage collector cover that ground.

### One dialect, one rule

Every difference from TypeScript obeys a single rule:

> Wherever TypeScript's types tell the truth, CoreScript computes the same values. Where they would have lied, CoreScript rejects the program, traps, or returns a typed absence. It never silently computes a different value.

| Compared with the same text in TypeScript | Allowed |
|---|---|
| CoreScript rejects what tsc accepts | Always |
| Same text, different static type | Yes |
| Same text and type, but a trap at run time | Yes |
| Same text and type, different value | Never |
| New syntax | Yes, and sparingly, since each addition forks the parser |

The reason is the agent loop. A compile error is cheap: the agent reads it and fixes the code. A silent difference, such as `7 / 2` evaluating to `3` because both operands happen to be integers, is the expensive kind of mistake.

The rule also gives most of the language an external oracle. Erase a CoreScript program the way TypeScript would and run it on Node: every run in which no check fails must produce the same result. The additions described below have no such oracle, so each needs its own specification and conformance tests.

What the dialect removes is run-time behavior that cannot be compiled to one meaning:

- Dynamic code and legacy modes: `eval`, `new Function`, `with`, sloppy mode, CommonJS, and computed `import()`.
- Operations that change an object's shape: `Proxy`, prototype mutation, `Object.defineProperty` on existing shapes, and `delete` on declared fields. Classes therefore have a fixed layout.
- `arguments`, and rebinding `this` across unrelated objects.
- Loose `==`, except `== null`, and object-to-primitive coercion.
- `WeakRef` and `FinalizationRegistry`, because finalization timing would differ between backends.
- Throwing values that are not `Error`s.

Nothing at the type level is cut. Conditional types, template literal types, and the rest of TypeScript's type machinery are erased before code generation, and npm's typings depend on them. Enums, namespaces, and parameter properties stay as well. A handful of constructs keep their spelling and change their rules:

- `any` is a box. It can be held and passed freely, and it must be narrowed before use; member access on `any` is rejected.
- `!` checks, and traps on every target.
- `readonly` is sound. A `readonly` property cannot be assigned to a writable one, and the checker enforces that. It is still a view, shallow and aliasable; immutable data is what records are for.
- `switch` must be exhaustive.
- `enum` loses reverse lookup, computed members, and declaration merging.
- `namespace` is a name path. A `Button.Props` interface beside a `Button` function, or a class nested under a class, are two definitions at two paths, and nothing merges at run time. Using a namespace as a dynamic bag (`Foo["Bar"]`, `Object.keys(Foo)`) is a compile error.
- `int` is a subtype of `number`. It is closed under `+`, `-`, and `*`, it traps outside the safe-integer range, and division returns `number`.

The checker is a fork of [tsrs](https://github.com/oxc-project/tsrs), a Rust port of the TypeScript 7 compiler. Declaration files from npm keep tsc's semantics, and authored code gets CoreScript's. We will track tsrs until its stable 7.1 release and then diverge for good, keeping one obligation: the declaration-file reader must keep up with tsc for as long as we consume npm typings, with tsrs's frozen baselines as its regression gate.

### Values: records, tuples, and structs

JavaScript objects have identity. Two objects with the same fields are different values, any holder of a reference can change the object, and no type says whether a function keeps or mutates what it was given. That is the wrong default for the data a shared core passes around. State, actions, effects, messages between actors, and anything handed to Swift or Kotlin must be comparable, hashable, and safe to send without asking who else holds a reference.

CoreScript adds value types with the syntax and semantics of TC39's withdrawn Records and Tuples proposal. `#{ x: 1, y: 2 }` is a record and `#[1, 2]` is a tuple. They hold only primitives, enum values, and other records and tuples, so a value is immutable all the way down, and `===` compares content. Objects written with `{}`, arrays, and class instances keep the meaning they have in JavaScript.

A `struct` declaration is to a record what a `class` is to an object. It names the type and gives it members: fields with defaults and `readonly`, getters for derived values, methods, and statics.

```ts
struct Money {
  readonly currency: Currency
  amount: int = 0

  get isZero(): boolean {
    return this.amount === 0
  }

  plus(other: Money): Money {
    return #{ ...this, amount: this.amount + other.amount }
  }
}

type Point = #{ x: number, y: number }
```

Declared structs are nominal. `UserId` and `OrderId` are different types even when each holds one string, and the declared name is the one Swift and Kotlin see. A record type written as a literal, like `Point` above, is structural and exact: a record with an extra field is a different type. Exactness keeps `===` meaning the same thing on every target, and it lets native backends store records inline because their size is known from the type.

A record literal takes the type its context expects, so `const origin: Point = #{ x: 0, y: 0 }` needs no constructor call, and `#{ ...other }` converts between types with the same fields. A struct with a private field cannot be written as a literal outside its own declaration. That is how a type like `Email` guarantees that every value of it passed validation.

Mutation follows the binding, as it does for structs in Swift.

```ts
const origin = #{ x: 0, y: 0 }
origin.x = 1          // error: `origin` is a constant

let cursor = origin
cursor.x = 1          // means: cursor = #{ ...cursor, x: 1 }
origin.x              // still 0
```

Assignment through a `let` binding produces a new value and rebinds the name. Nothing is mutated in place as far as the program can observe, so no alias can see the change. A `readonly` field rejects the assignment even through `let`. A reducer can therefore read like mutation and still return a new state.

We do not extend this rule to plain objects, arrays, or class instances, although Swift's guarantee is attractive enough that we looked hard for a way. A constant binding cannot stop writes through another alias, so `const` would promise less than it appears to. Copying on assignment would fix that and silently change what existing TypeScript code computes. And the rule needs to know which methods mutate their receiver, which DOM and npm typings do not say. The practical route to more local reasoning is to make structs good enough that most data is a struct, and to keep classes for things that have identity: resources, views, and services.

The web is where values cost something. JavaScript's `Map` and `Set` compare objects by identity, so on that target the compiler must emit equality and hashing for records, or intern them.

### Enums and pattern matching

TypeScript programs spell a closed set of alternatives as a union of object types with a shared literal field, called `kind` in one codebase, `type` in the next, and `tag` in a third. CoreScript has one canonical form: an enum whose cases can carry payloads. The base is the TC39 enum proposal, a subset of TypeScript's enums with iteration in place of reverse lookup. We keep auto-numbering, which that proposal drops, and we add the payloads it lists as future work.

```ts
enum LoadState {
  Idle,
  Loading,
  Loaded(items: #[Item]),
  Failed(error: LoadError),
}

const label = match (state) {
  when LoadState.Idle: "Nothing loaded yet";
  when LoadState.Loading: "Loading";
  when LoadState.Loaded(let items): `${items.length} items`;
  when LoadState.Failed(let error): error.message;
}
```

`match` is an expression, and it must be exhaustive. Adding a case to `LoadState` stops every `match` that lacks it from compiling, which turns "find every place that handles this" from a search into a list of errors. The `is` operator tests a single pattern. Case patterns such as `LoadState.Loaded(let items)` resolve statically to the enum's declaration. We leave out user-defined matchers, which the TC39 proposal builds on a `Symbol.customMatcher` protocol looked up at run time.

Two smaller proposals come along because they make expressions compose. A `do { }` expression evaluates to its last expression, so a `match` arm can hold statements. The pipeline operator is pure desugaring that lets standalone functions chain the way methods do. Its placeholder token is undecided upstream and undecided here.

### Checked casts and traps

TypeScript's `as` is an unchecked assertion. `value as User` compiles whether or not the value is a `User`, and nothing happens at run time. CoreScript splits the operator in three, parallel to Swift's and to the `try` family.

- `as` keeps TypeScript's meaning wherever that meaning is safe. Every downcast that tsc accepts becomes a compile error that points at the two checked forms.
- `as?` checks at run time and evaluates to `T | undefined`.
- `as!` checks at run time and traps if the check fails.

The target of a checked cast has to be something the runtime can test, so function types and bare type parameters are rejected, and the target of `as?` cannot include `undefined`, since failure would be indistinguishable from success. `as const` and `satisfies` are unchanged. Both new spellings are errors in TypeScript today, so no existing program changes meaning.

A trap is a bug in the program, and it sits deliberately outside the error system. A failed `as!`, a failed `!`, and `int` overflow all trap. Traps never appear in a `throws` clause, and `catch` does not catch them on any target.

### Actors and isolation

CoreScript's concurrency model is the actor model as Swift implements it. An `actor` declares isolated state. Code inside the actor runs serially and reads that state synchronously; code outside reaches it with `await`. The definition is about isolation and says nothing about threads. Where an actor runs is the business of an executor: on native targets actors share a thread pool, and on the web many actors can share one web worker. Changing that placement changes no signature.

```ts
actor Session {
  private token: Token | undefined

  async refresh(): Promise<Token> throws(AuthError) uses(network) {
    this.token = try await requestToken()
    return this.token
  }
}

const token = try await session.refresh()
```

Only immutable data crosses between actors: records, tuples, primitives, enum values, and references to actors. Passing a mutable object or a closure across is a compile error. Values make message passing cheap, and they make the rule simple enough to check from types alone. The same rule covers the other two boundaries a shared core has, between client and server and between CoreScript and Swift or Kotlin.

There is a predefined main actor. Top-level mutable bindings belong to it, which gives ordinary single-threaded code the meaning it has today and makes any other actor `await` to touch them. Platform toolkits that must be called on the main thread, such as UIKit and Android views, are imported as isolated to it.

Isolation is the third kind of effect metadata. A function that synchronously touches an actor's state is inferred `isolated` to that actor, so the annotation Swift programmers write by hand is mostly computed here. The clause names an actor, never an executor: `isolated(Main)`, `isolated(account)` for a parameter, and an implicit `isolated(this)` on an actor's methods. A function holds at most one isolation. Touching two actors synchronously in one body is a compile error that points at `await`.

Where a function runs is readable from its signature:

- A synchronous function runs in its caller's domain. It cannot suspend, so it cannot move.
- A method on a class runs where the instance lives, and a closure runs where it was created, because neither can cross.
- An async function with an `isolated` clause runs on that actor.
- A free async function with no clause runs in no actor's domain. Natively that is the pool.
- An async function that takes mutable arguments or closures says `isolated(caller)`, which is never inferred.

The fourth rule is Swift's original one, from [SE-0338](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0338-clarify-execution-non-actor-async.md). Swift 6.2 moved away from it in [SE-0461](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md), so that async functions run on the caller's actor by default. We keep the original on purpose. Under it, a mistake is a compile error at the call, and the error names the fix. Under the newer default, the mistake is heavy work running on the main actor, which no checker reports. Swift changed in part because the old default made async functions that take non-sendable values unusable from actors, and `isolated(caller)` covers that case explicitly. We expect the benchmark to confirm this choice or overturn it.

We do not adopt TC39's structs or shared structs. Classes already have a fixed layout, and shared structs permit data races by design.

### Compile-time evaluation and decorators

Effect metadata tells the compiler which code is safe to run at build time. Any expression whose `uses` set is empty can be evaluated by a compile-time interpreter, and `comptime expr` asks for exactly that. The result has to be a value the codebase can store: a primitive, a record, a tuple, or an enum case. A throw or a trap during evaluation is a compile error. Results are cached by the hash of their inputs, and evaluation is limited by step count, since a wall-clock limit would make a build's success depend on the machine. Compile-time code may read the codebase itself, including stored assets such as an OpenAPI document, because those inputs are content-addressed and become part of the cache key. It gets no clock, randomness, network, environment, or ambient files.

Decorators are the same mechanism applied to a declaration. CoreScript keeps the TC39 decorator syntax and semantics with two restrictions: the decorator expression must resolve statically to a known definition, and running it must stay within what compile-time code may do. The function a decorator returns may use capabilities, and those are added to the decorated member's `uses` clause.

Applying a decorator is a typed function call, so the decorated member takes the type of the call's result. TypeScript does not allow this today, and it is useful: a decorator can turn `T | undefined` into `T` because of a check it performs.

```ts
function nonNull<A extends unknown[], R>(
  method: (...args: A) => R | undefined,
  context: ClassMethodDecoratorContext,
): (...args: A) => R {
  return (...args) => method(...args)!
}
```

The cost is that the signature as written is no longer the signature callers see. The body is checked against what is written, callers are checked against the result, and any view of a contract must show the effective signature. The expansion is stored by hash, so showing it is cheap.

A second kind of decorator receives the checked declaration as data and returns new declarations: a generated totality test for a reducer, lens helpers for a state type, a typed client from a service contract. It avoids the problems Swift's macros have because of how they are built. A decorator is an ordinary definition run by the compiler's own interpreter, so there is no syntax library to version and no separate plugin to build. It receives types where a Swift macro receives only syntax. Its expansion is cached, viewable, and diffable. We plan the wrapping kind first and the deriving kind after it.

## Verification

### Who checks what

A single implementation divides the work of trust. Whether the platforms agree is the compiler's question, answered deterministically. Whether the code meets the requirements is the question that agents and people spend their judgment on.

- The compiler checks types soundly, with every escape hatch visible in a contract and in review. It checks effect clauses on exports, exhaustive `match` and `switch`, sound `readonly`, and that no export reaches a capability its target lacks.
- The host compilers check that presentation code handles every exported state.
- Tests check behavior against requirements, and their results are evidence about exact hashes.
- People judge intent, and they judge presentation, where nothing is guaranteed to agree across platforms. The goal is to remove the unnecessary duplication that exists today, and some duplication is the price of native interfaces.

### Oracles

Every claim of correctness needs something outside the implementation to compare against. For the TypeScript-compatible part of the language, that is Node, through the erase-and-run rule, with test262 defining the engines' behavior. For the standard library, it is Web Platform Tests and Node's test suite. For a package, it is the package's own tests on each backend. The dialect's additions have no external oracle, and each one needs a specification and conformance tests before it ships.

### The cold-agent benchmark

Many choices in this document are claims about what agents read and write well, and CoreScript starts with no training corpus of its own. Both can be measured. Give an agent that has never seen the language only its documentation and its tools, then compare turns to a passing build and defect rate against TypeScript, Swift, and Kotlin on a fixed set of tasks. The tasks should include editing existing code as well as writing new code, and each run should be repeated with and without the agent-first tooling described above.

Because code is stored as trees, a syntax variant costs a parser and a printer. The benchmark can therefore decide questions that would otherwise be settled by taste: tags with quoted text and plain control flow against JSX with marked control flow, the required `try`, the pipeline operator, and the default isolation of async functions. We should build it early. It has not been run. Until it has, the claim that agents can work in a dialect they were not trained on rests on argument alone.

The benchmark also has to beat a baseline that needs no new language at all: a precise specification plus an oracle, with agents regenerating native code for each platform. If that turns out to be as reliable as one shared implementation, CoreScript is solving a problem that has gone away.

## What is not in this vision

- Shared presentation. Interfaces are built per platform, with the platform's toolkit.
- Ownership, borrowing, or lifetime annotations.
- A JavaScript engine embedded in native apps as the way shared code runs, and WebAssembly as the way it reaches the web.
- A central package registry, and source control that depends on files.
- Staying compatible with TypeScript as it evolves. After the fork point, new TypeScript features are adopted only when we want them.
- A `const` that makes class instances deeply immutable. A weaker, inferred read-only view is possible later if evidence asks for it.
- Named global actors beyond the main actor.
- Compile-time code that produces types. Generating declarations belongs to deriving decorators.

## Open questions

The language:

- Whether classes written in CoreScript are nominal, as structs are. Fixed layouts argue for it.
- The keyword for actors. `worker` was the original proposal and `actor` is used here.
- How the required `try` marker reads inside JavaScript's `try { } catch { }` statement.
- Whether a record may hold a reference to an actor.
- A policy for `null` and `undefined`.
- The file extension for materialized files, which needs replacing now that the language has a new name.
- The design of enum payloads, the pipeline token, and deriving decorators.

The codebase, source control, and registries:

- The hashing scheme, and the editing workflow for agents and for people.
- Every mechanism of source control: the draft tier, landing, permissions, and how publication gates are expressed.
- The registry's record formats, where code is hosted, and whether a package is its own account.
- How one checker runs two modes when a rule such as sound `readonly` lives in the type relation itself.

The tools:

- The name of the command, written `core` in this document.
- The schema for commands and for diagnostics, and how much of the language-intelligence surface is commands and how much is views.

Backends and runtime:

- The native code generator. None has been chosen.
- The Windows and server backends.
- Writing Kotlin metadata from a compiler that is not Kotlin's. This needs a spike, and the fallback is a thin generated Kotlin overlay.
- Async in a native library that the host app drives, which is the first hard runtime problem.
- Cycle collection, the cost of an `int` overflow check per operation on the web, and equality code against interning for records there.
- The order in which to build all of this.

Packages from npm:

- Whether a package that neither conforms nor ports can pull in an embedded engine, with the build naming the package that forced it.
- Checking values at the boundary of an unverified declaration file against recording the trust.
- How sound `readonly` meets typings written without it.

## Alternatives considered

### Kotlin Multiplatform

It shares code across the platforms we care about and has years of production use. The code it generates for JavaScript and Wasm is less than optimal, and Kotlin's defaults hide meaning that an agent has to carry in its head and a reviewer cannot see. We also do not want the Kotlin, Gradle, and JVM toolchain as a requirement for every platform.

### A new language

Our first design was a new language with a Swift-shaped surface, since Swift already has typed `throws`, marked call sites, enums with payloads, value types, and argument labels. It would have started with no corpus and no oracle, and it would have spent the project's effort on syntax. The features that made Swift attractive are in this document as additions to TypeScript.

### TypeScript as a strict subset

Removing features without adding any would have kept every existing tool working unchanged. It leaves no way to state failure or capabilities except by convention, such as errors as return values and capabilities as parameters, which changes how every function is written. Accepting that CoreScript is a dialect reopened the features we had ruled out for changing the language too much.

### Emitting Swift and Kotlin source

Transpiling shared code to each host language would make the output look native. Every construct would then have to type-check and mean the same thing in three languages, which makes the smallest of the three the real language. We emit native code and class files and keep the host language at the seam.

### A Rust core with a shared shell

Projects such as [NSH](https://github.com/longbridge/nsh) pair a Rust core with a rendering shell that runs everywhere. That shares presentation, which we chose not to share, ships about a megabyte of runtime to the web, and asks for a stricter language than consumer applications need.

### An embedded JavaScript engine on mobile

Running the core in JavaScriptCore, Hermes, or QuickJS would work on the first day and would make every npm package available. It also ships a second runtime and a second heap inside every app, which is the opposite of the goal.

### Git and a central registry

Keeping files in git and publishing to npm would let every existing tool keep working. Both are built on files and on trust in a host. Definitions stored by hash need neither, and the tools we would give up are the ones this vision replaces.

## Prior art

CoreScript borrows more than it invents.

- [Unison](https://www.unison-lang.org/) for content-addressed code and for the distinction between structural and unique types.
- [tcm](https://tcm.codes/) for content addressing applied to TypeScript, the update list, definition-level version control, search by type, and its AT Protocol registry.
- [Zena](https://zena-lang.dev/) for its package manager design on the AT Protocol, and for [its essay on languages for the AI era](https://zena-lang.dev/blog/2026/09/languages-for-the-ai-era/), whose argument that agents can work in a language they were not trained on is what the benchmark exists to test.
- The [AT Protocol](https://atproto.com/) for portable identity, signed repositories, and independent indexes and labelers.
- [Veil](https://github.com/RKKatakam/veil), [jj](https://github.com/jj-vcs/jj), and [Scion](https://www.anthropic.com/research/redesigning-source-control) for source control built for parallel agents.
- Cloudflare's `cf` command line, by way of the guide in [docs/agent-first-clis.md](docs/agent-first-clis.md), for the terminal interface.
- [tsrs](https://github.com/oxc-project/tsrs) for the checker we fork.
- [scriptc](https://github.com/vercel-labs/scriptc) and [Porffor](https://porffor.dev/) for evidence that TypeScript compiles ahead of time to native code, and for testing each program against Node.
- Swift for value semantics, the `as` and `try` families, typed throws, actors, result builders, and the lessons of its macros and of its concurrency defaults.
- TC39 proposals for [records and tuples](https://github.com/tc39/proposal-record-tuple), [pattern matching](https://github.com/tc39/proposal-pattern-matching), [do expressions](https://github.com/tc39/proposal-do-expressions), the [pipeline operator](https://github.com/tc39/proposal-pipeline-operator), [decorators](https://github.com/tc39/proposal-decorators), [extractors](https://github.com/tc39/proposal-extractors), and enums.
- TSRX and the [JSX 2.0 discussion](https://github.com/react/jsx/issues/65) for markup.
- The Elm Architecture for the shape of an application.
