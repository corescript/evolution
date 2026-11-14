# A Vision for CoreScript

> This is a prospective vision document. Nothing described here is implemented yet. It records the goals of CoreScript and the basic approach to reaching them, in enough detail to argue with. The language itself is described in [DESIGN.md](DESIGN.md). Concrete designs will be proposed and reviewed one at a time through the CoreScript evolution process, and that review may reject or revise what is sketched here. All syntax and every command shown are illustrative.

## Introduction

A product that ships as a server, a web app, an iOS app, and an Android app has four implementations of much of its logic and four sets of tests, written in three or four languages, and all of them must agree. Making the server authoritative removes some of the duplication, and generating types removes a little more. Validation, state machines, formatting, navigation, caching, and offline behavior are still written once per platform. The copies drift, and they drift faster now that products change every week.

Agents write most of that code. The cost of writing it has collapsed, and the cost of knowing it is right has not. Humans review evidence more often than they review code: preview builds, screenshots, test results, changed contracts. With four implementations, the question a reviewer cares about, whether the code meets the requirements, is tangled up with a second one, whether the implementations agree. The second question is mechanical. A shared language answers it once, as a matter of compiler conformance, and leaves human and agent judgment for the first.

Some of that duplication is the price of doing multiplatform apps well. A product should be built with the toolkit native to each platform: the DOM on the web, SwiftUI, UIKit, and AppKit on Apple platforms, Jetpack Compose and `android.view` on Android, WinUI on Windows, GTK on Linux. Each platform has its own conventions and capabilities, and an interface that ignores them feels foreign. So the views, and the interactions inside them, are written once per platform. Nothing else has a reason to differ: business rules, the state machine behind each screen, service contracts, navigation, validation, and formatting.

CoreScript is one programming language for the whole product, together with the tools an agent needs to work in it. Our aim is that it is the only language we and our agents write when building for consumers. Everything shared is written once. Each platform's views are written separately, in the same language and with the same declarative model, against that platform's own toolkit. To make that possible, CoreScript compiles to what each platform's frameworks expect: JavaScript on the web, code that follows Swift's ABI on Apple platforms, and class files that Kotlin's runtime understands on Android.

The language is a dialect of TypeScript. It keeps TypeScript's syntax and type system, removes the dynamic behavior that cannot mean the same thing on every platform, and adds value types, nominal protocols, checked casts, effect clauses, and actors. Its tools assume that the author is an agent and the reviewer is a person looking at evidence.

Eight things define CoreScript, and the main sections of this document take them in order:

- A [content-addressable codebase](#a-content-addressable-codebase). Code is stored as definitions keyed by hash, and text is a view.
- [Effect metadata](#effect-metadata). Every definition carries what it can throw, which capabilities it uses, and where it must run.
- [Distributed package registries](#distributed-package-registries). Releases are signed records on the AT Protocol, and anyone can run an index.
- [Agent-first source control](#agent-first-source-control). History, merging, and publication work on definitions and are built for many agents at once.
- An [agent-first terminal interface](#an-agent-first-terminal-interface). One schema-driven command line that an agent can discover, preview, and verify with no prior training.
- [Agent-first language intelligence](#agent-first-language-intelligence). The compiler answers questions about meaning as structured data.
- [Multi-target compilation](#multi-target-compilation). JavaScript for the web and for servers, native code that follows Swift's ABI on Apple platforms, and class files for Android, with Windows and Linux to follow.
- [Multi-toolkit user interfaces](#multi-toolkit-user-interfaces). One declarative model for views, written once per toolkit: the DOM, UIKit, AppKit, `android.view`, GTK, WinUI, SwiftUI, and Jetpack Compose.

The closest existing answer is Kotlin Multiplatform. The code it generates for JavaScript and Wasm is less than optimal, and Kotlin carries too much implicit meaning and too little verification for code that agents write and people skim. Its interface story also forces a choice: share one Compose interface that is drawn the same way everywhere, or write each platform's views in that platform's language. CoreScript's bet is one language, one implementation of everything shared, native views on every platform, and evidence attached to exact code.

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

Principle 4 asks for one meaning everywhere, and principle 5 asks for the host's own primitives on each platform. That tension is the project. We resolve it by giving shared code one meaning, built on JavaScript's semantics, and by writing each platform's presentation separately, in the same language, against that platform's own toolkit.

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

There is a lossless canonical text form. Review and editing need one, and it makes a git mirror of a codebase cheap. When files have to be materialized, they use the `.core` extension, and they are a rendering of the store and never its source of truth.

### Changes that carry through

References by hash change what a conflicting edit looks like. Suppose one agent changes a function's signature while another adds a caller. Git merges the two cleanly, and the build breaks. Here the new caller stays valid against the old definition, and it appears on a list of things to update when the name moves. Dependents that still type-check follow a change automatically. Those that do not keep their old version and go on the list, so nothing is ever half-updated. The mechanism is tcm's, and CoreScript leans on it: a function that starts throwing puts every caller that lacks a `try` on that list.

### Evidence by hash

Principle 2 asks that every verification result belong to exact inputs, and hashes are how the system knows what the inputs were. A test result is recorded against the hashes it exercised, and when the test has no effects, it is a pure function of those hashes and can be cached indefinitely and shared between machines. Decorator expansions and compile-time results are cached the same way. We intend evidence in general to be recorded as a verdict about a specification hash, an implementation hash, and a checker hash, so that deciding whether a candidate is fit to publish becomes a query.

### Other formats in the codebase

Part of a product's meaning lives in formats that are not CoreScript: a SQL schema, an OpenAPI document, a stylesheet, a page of Markdown. TypeScript 7.1 introduced content mappers for this. A mapper is registered for a kind of file and produces a typed representation of it, with positions mapped back to the original, so types flow out of the foreign format and diagnostics land inside it. The checker we fork implements them, and CoreScript keeps them.

A stored asset is content-addressed like a definition, so a mapper's output can be cached by the asset's hash. TypeScript runs a mapper as an external process and asks for explicit permission before it does. We expect a CoreScript mapper to be compile-time code with no capabilities, which needs no such permission.

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
| `throws(E)` | the call may fail with error `E` | a call marked `try` | `catch` |
| `uses(C)` | the call needs capability `C` | any call | a scoped handler |
| `isolated(A)` | the call must run in actor `A`'s domain | a synchronous call | `await` from another domain |

### Failure

A call that can throw must be marked, with three forms: `try f()` propagates the error, `try? f()` evaluates to `T | undefined`, and `try! f()` traps. The marker is required for two reasons. A stored codebase has one spelling per meaning. And a required marker makes a contract change loud: when a function starts throwing, its callers stop compiling and land on the list of things to update. With every throwing call marked and every `throws` clause known, `catch (e)` binds a real type, the union of what the block it guards can throw, where TypeScript gives `unknown`.

### Capabilities

`uses` names the capabilities a function needs: the network, the clock, randomness, the file system, a database. Code still calls `fetch` and `Date.now()` directly. The clause records that it did. Tests replace a capability with a scoped handler, which every runtime has to carry across `await`. A capability can have several implementations, so `uses(d1)` can be Cloudflare's database on a server, SQLite on a phone, and a fake in a test, as long as the capability's contract promises only what all of them provide.

### Isolation

`isolated` names the actor whose state a function touches. It is described with the rest of the concurrency model in [DESIGN.md](DESIGN.md#actors-and-isolation).

### What the metadata is for

The clauses are the visible part. The metadata exists for every definition, and a good deal of the system reads it.

- Availability and placement. An export exists on a target when every capability in its `uses` set is implemented there. Code that reaches for the file system is a compile error in a browser build and compiles everywhere else, with no separate availability annotation to maintain.
- Sound caching. A test whose `uses` set is empty is deterministic, so its cached result can be trusted.
- Compile-time evaluation. An expression with no effects is safe to run during a build.
- Review. A change is summarized in terms of contracts, such as "this change adds `network` to `checkout`".
- Policy. Tooling can enforce rules per namespace, for instance that nothing under a domain layer uses the network.
- Supply chain. A library's effects are visible before anyone depends on it, and a release that starts using the network shows up as a changed contract.
- Architecture. A reducer declared with no effects is checked to have none, which turns "no clock, no random, no I/O" from a convention into a compile error.

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

npm remains the source of foreign code, handled as described under [Packages](#packages). The AT Protocol registry is for libraries written in CoreScript. Its design is still to be done.

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

The commitments above are firm. The mechanisms behind them are still to be designed.

## An agent-first terminal interface

With no files, the command line is the way into the codebase. Adding a definition, rendering a view, running the tests a change reaches, building for a target, and publishing a release are all commands of one program, `core`. The design follows the guide in [docs/agent-first-clis.md](docs/agent-first-clis.md), whose test is that a capable agent with no prior knowledge of the tool can use it with no special prompting.

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

Programs get the same access. TypeScript 7.1 restored a compiler API that tools reach over a protocol, and the checker we fork implements it. CoreScript keeps that API and widens it to the queries and edits above, so a harness, an editor, or a review tool asks the compiler directly.

All of this is served from the same schema as the terminal interface, and over MCP. A conventional language server remains available for people in editors, since the checker we fork includes one.

## Multi-target compilation

One codebase compiles to every platform a product ships on, and each target gets output native to it. On the web that is JavaScript. On native targets it is the platform's own binary format, produced directly: a CoreScript app carries a small runtime of its own, with no JavaScript engine inside, and its code never passes through Swift or Kotlin source on the way.

| Target | Output | Memory | Stage |
|---|---|---|---|
| Web | JavaScript modules, with nothing to download beyond inserted checks and value equality | The engine's garbage collector | First release |
| Server | JavaScript on Node, Bun, and workerd, where any npm package also runs | The engine's garbage collector | First release |
| Apple platforms | Native code generated through LLVM, following Swift's ABI | Reference counting with a cycle collector | First release |
| Android | JVM class files that D8 and R8 turn into DEX, with the metadata Kotlin reads | The host's garbage collector | Later |
| Windows and Linux | Undesigned | Undecided | Later |

The same rules apply on every target. A program that compiles for one target means the same thing on all of them, and the web build carries the same inserted checks as the native ones. The memory rule is that a process never holds two garbage collectors: on Apple platforms the runtime counts references and runs a cycle collector for what counting cannot free, and on Android it uses the collector that is already there. Finalization timing is unobservable everywhere.

### Speaking each platform's ABI

To a platform's frameworks, CoreScript's output has to be indistinguishable from code written in the platform's own language. That is what lets a CoreScript type conform to a SwiftUI protocol, or a CoreScript function act as a Compose composable, and it asks more of the compiler than being callable from the host language would.

On Apple platforms it means Swift's ABI: its calling conventions, type metadata, value and protocol witness tables, and symbol names, for generic code as well as concrete code. Native code is generated through LLVM, which Swift's own compiler uses, so the conventions that ABI depends on are already there. On Android it means class files with the metadata Kotlin reads, following the conventions of Kotlin's coroutines and of the Compose runtime. CoreScript writes all of this itself, with no Swift or Kotlin overlay between its output and the platform.

The mapping between CoreScript's types and each platform's is fixed, and it holds in both directions.

| CoreScript | Swift | Kotlin |
|---|---|---|
| struct, record, tuple | struct | data class |
| enum | enum | enum class |
| enum with payloads | enum with associated values | sealed interface |
| protocol | protocol | interface |
| `int` | `Int` | `Long` |
| async function | async function | `suspend` function |
| parameter names | argument labels | named parameters |

Because each side sees the other as native, a product with existing Swift or Kotlin code can adopt CoreScript one module at a time. The destination is a product written entirely in CoreScript.

### Calling the platform

The codebase is its own declaration format. A library release is a set of signatures and hashes, and nothing separate has to be generated or kept in sync with it. Foreign APIs enter through importers that translate a platform's declarations into CoreScript signatures, a whole framework at a time:

- TypeScript declaration files, for npm packages and the web platform.
- C and Objective-C headers, read through Clang, which covers UIKit, AppKit, and Foundation.
- Swift module interfaces, the textual descriptions Apple ships in its SDKs for frameworks such as SwiftUI.
- JVM class files together with Kotlin's metadata, which covers `android.view` and Jetpack Compose.
- Windows metadata files, which describe WinRT, where WinUI lives, and, through Microsoft's win32metadata project, Win32 and COM.
- GObject Introspection data, which describes GTK.

Each platform's nominal interfaces arrive as CoreScript [protocols](DESIGN.md#protocols): Swift and Objective-C protocols, JVM interfaces, WinRT interfaces, and GObject interfaces. A type conforms to one by declaring it, which is how a CoreScript type becomes a SwiftUI view or a UIKit delegate. TypeScript's interfaces stay structural.

Foreign formats state some of what a CoreScript signature carries and miss the rest. Swift interfaces already say which functions throw, which are async, and which are isolated to the main actor. No format states capabilities, and several get nullability wrong. API notes supply the difference. A note is a sidecar keyed to foreign symbols, so it survives a framework or package update where a forked declaration would conflict; Apple's API notes for Objective-C are the precedent. For npm packages, whose source is JavaScript or TypeScript, the compiler reads the source and infers effects as it does for code written in CoreScript, and notes cover the remainder. Platform capabilities are declared by hand at the bottom: the binding written over `NSURLSession` is where `uses(network)` is defined.

### Packages

An npm package is foreign code, in the way UIKit is. The dialect governs what you author and says nothing about a dependency's internals. On the web and on a server, any package runs on the engine as it always has. Its declaration file is a claim nobody verified, so values arriving from it are either checked at the boundary or accepted with the trust recorded.

Inside shared code, on native targets, a package has to be compiled. One that conforms to the dialect compiles as is. For one that does not, the usual answer is to port the slice the product uses, with the package's own tests as the oracle; the [calibration brief](docs/agentic-dev-calibration-brief.md) documents how cheap that has become. A package's test suite passing on each backend is the evidence that it works there.

### One standard library over a small native floor

CoreScript offers the web-standard APIs on every target: `fetch`, streams, `URL`, and the rest. They are the canonical library. Node's APIs are provided as a compatibility layer for npm packages, since they are defined by one implementation's behavior. A native app needs these APIs and has no use for a JavaScript engine to run them, so we reimplement them and embed nothing.

The API layers are written in CoreScript over a small native floor per platform: the host's HTTP stack or sockets, files, the clock, randomness, and cryptographic primitives. The layers compile once for every backend. Web Platform Tests and Node's own test suite verify them once, and each platform has only its floor to verify. Availability differs between targets and meaning does not.

Service-shaped platform APIs fit the same model. A key-value store or a SQL database in the style of Cloudflare's Workers bindings is a capability with several implementations: the hosted service on a server, SQLite on a device, and a fake in a test.

### Values in memory

A record's layout is known from its type, so native backends keep records on the stack, in registers, or inline in a parent record, class field, or array, with no header and no reference count. Assigning to a field through a `let` binding compiles to a store into one slot. Reference counts remain only on the heap storage a record can point to: strings, variable-length tuples, recursive records, and records large enough that the compiler chooses to box them. That storage is immutable and may cross between actors, so we expect its counts to be atomic, while class instances stay inside one actor and can use cheaper non-atomic counts. On the web, records are ordinary garbage-collected objects.

## Multi-toolkit user interfaces

CoreScript shares everything except the views and the interactions inside them. A view is written for one toolkit, in CoreScript, and rendered by that toolkit's own machinery. What carries from platform to platform is the programming model: the same language, the same architecture, and the same way of composing a view from state. Skills, patterns, and much of the code are reused, and each view still follows its platform's conventions and uses its capabilities.

React Native named this approach in 2015 as "learn once, write anywhere", and Apple made the same promise for SwiftUI across its own platforms. CoreScript extends it across toolkits from different vendors.

| Toolkit | Platform | Style | Declarations come from |
|---|---|---|---|
| DOM | Web, with rendering on the server | Imperative | TypeScript declaration files |
| UIKit | iOS and its relatives | Imperative | Objective-C headers |
| AppKit | macOS | Imperative | Objective-C headers |
| `android.view` | Android | Imperative | Class files |
| GTK | Linux | Imperative | GObject Introspection data |
| WinUI | Windows | Imperative | Windows metadata |
| SwiftUI | Apple platforms | Declarative | Swift module interfaces |
| Jetpack Compose | Android | Declarative | Class files and Kotlin metadata |

A product picks the toolkits it wants. Two views of one feature on two toolkits share their state, their actions, their logic, and their tests. They differ where the platforms differ: which elements exist, how layout works, how navigation is presented, and which gestures people expect.

### One architecture under every toolkit

The language is designed against a specific shape of application, set out in [docs/ui-architecture.md](docs/ui-architecture.md). It is The Elm Architecture with its informal parts made checkable. Its one rule is that nothing is ambient: every input to behavior is a declared value or a function of state. Most of that document turns into something the compiler checks.

- State is a struct. Stored fields are fields and derived fields are getters, which cannot be assigned by construction.
- Actions are enums with payloads. The three provenances the spec names (intents from the view, completions from effects, emissions from subscriptions) are three nested enums, and a view's `dispatch` accepts only intents.
- `Update` is a function with no `uses` clause and no `throws` clause. Its body is one `match` over the state and the action, with clauses tried in written order, and the stored tree can be rendered as the spec's transition table for review.
- Effects are records. Tests compare them by value, and an effect's completions, key, and concurrency policy are static declarations.
- Lenses need no library. A state lens is a field plus the `let` update rule, and an action prism is a case pattern.
- The runtime shim that interprets effects against services is an actor.
- A view receives a state value and a `dispatch` function, on every toolkit.

A pure `Update` has consequences beyond tidiness. Its tests can be cached soundly, a recorded sequence of actions replays exactly, and an agent can drive a component with no interface attached by dispatching actions and inspecting state. Everything above the view has one implementation, so the spec's per-platform clauses shrink to the view, the service interpreters, and the conventions for mount and cleanup.

State is a value, and rendering is a pure function of it. The language does not force that choice on a library, because the builder decides how holes are kept current: one builder can diff hole values between renders, and another could subscribe them to signals.

One change to the spec follows from this design. Subscriptions are currently `Observable<Action>[]`, which cannot be compared. They should be descriptions, like effects, so that they diff and test the same way.

### One model for views

A view renders a state value as a builder block. Inside a block, written `@{ }`, every statement contributes to the result: tags create elements, text is an ordinary quoted string, and `if` and `for` are the language's own. [DESIGN.md](DESIGN.md#markup) gives the rules. The tags are the toolkit's own types, so a view for the web is written with HTML elements:

```ts
function CartView(state: CartState, dispatch: Dispatch<CartIntent>): Html {
  return @{
    <h1>"Cart"</h1>
    if (state.items.length === 0) {
      <p>"Your cart is empty"</p>
    } else {
      <ul>
        for (const item of state.items; key item.id) {
          <li>item.name</li>
        }
      </ul>
      <button onclick={() => dispatch(CartIntent.CheckoutTapped)}>"Check out"</button>
    }
  }
}
```

A view for SwiftUI is written with SwiftUI's views:

```ts
function CartView(state: CartState, dispatch: Dispatch<CartIntent>): SwiftUI.View {
  return @{
    <NavigationStack>
      if (state.items.length === 0) {
        <ContentUnavailableView title="Your cart is empty" systemImage="cart" />
      } else {
        <List>
          for (const item of state.items; key item.id) {
            <Text>item.name</Text>
          }
        </List>
        <Button action={() => dispatch(CartIntent.CheckoutTapped)}>"Check out"</Button>
      }
    </NavigationStack>
  }
}
```

`CartState`, `CartIntent`, the update function behind them, and its tests are the same definitions in both. An agent that has built the first view already knows how to build the second, apart from the toolkit's vocabulary.

These examples write a view as a function. Whether views are functions, structs, or something else is an open question. So are the way Swift's argument labels and modifiers are written as attributes, and the way a build selects between two definitions of one view.

### Imperative toolkits: bindings and a builder

The DOM, UIKit, AppKit, `android.view`, GTK, and WinUI are trees of objects that a program creates and then mutates. For each, the importer generates typed bindings to the whole framework, so CoreScript can drive the toolkit directly whenever it needs to. A builder library for the toolkit then implements the declarative model on top of the bindings.

Markup for these toolkits lowers to static calls on the builder, and the calls carry the structure the compiler already knew. Each block of static structure becomes a template, a compile-time constant with a content hash, and a render is template identities plus the values in their holes. The builder creates a template's native objects once and afterward writes only the holes that changed: a label's text, a row inserted for a new key, a subtree swapped when an `if` takes its other arm. View identity comes from the same structure, and because template hashes are stable across builds, hot reload and hydration can match the views that did not change. On the web the same builder renders HTML on the server and adopts it in the browser.

These builders are ordinary libraries, written in CoreScript and held to the same verification as any other code, as principle 8 asks.

### Declarative toolkits: direct translation

SwiftUI and Jetpack Compose are already declarative. Each has its own runtime for identity, state, diffing, and layout, and a second reconciler stacked on top would work against it. For these toolkits a builder block is translated directly into the toolkit's own form.

A block that targets SwiftUI compiles to the same generic view values that Swift's compiler would build from an equivalent view builder. A tag is a call to that view's initializer, an `if` becomes SwiftUI's conditional content, and a keyed `for` becomes a `ForEach`. The result has a concrete Swift type assembled from SwiftUI's generics, and SwiftUI lays it out and diffs it as it would any Swift view. This is where following Swift's ABI for generic code is required.

A block that targets Compose compiles to composable calls. Compose's compiler plugin rewrites every composable function to receive a composer and a bitmask describing which arguments changed, and wraps its body in groups that the runtime can skip or re-run. CoreScript's Android backend performs the same rewrite. Records suit it, because they are immutable and compared by value, which is what Compose relies on to skip a call.

The architecture meets these toolkits through their own observation mechanisms. The store publishes each new state the way the toolkit expects to hear about changes, and the toolkit decides what to redraw.

Both translations depend on contracts that their owners wrote for their own compilers, and they are the largest technical risk in this vision.

- In the macOS 27.2 SDK, SwiftUI's interface marks about 670 declarations, and SwiftUICore's about 330, so that no symbol is exported and every client compiles the body itself. `ViewBuilder.buildBlock` is one of them. The build calls `swiftc` for these, in the way it calls Clang for headers. It hands Swift's compiler a small generated file that wraps the declarations a program uses, and links the code that comes back. That file is a build artifact that nobody reads or edits. A trial with `buildBlock` and `buildEither` produced generic functions that follow Swift's calling convention.
- The checker has to model enough of Swift's type system to type SwiftUI: generics constrained by protocols, associated types, opaque result types, and overloads chosen by argument label.
- The convention between Compose's compiler plugin and its runtime is theirs to change, and at least one past change to it required every library to be recompiled. Both are open source, so CoreScript tracks upstream's implementation release by release.

Each is the subject of an early spike.

## The language

[DESIGN.md](DESIGN.md) describes the language. This section says where it starts and why.

CoreScript starts from TypeScript. Agents already write it well, from a very large corpus. It has mature tooling, including dozens of Rust crates. On the web it is the language itself, and JavaScript engines and test262 already define what a correct program does. Other projects have shown the remaining pieces work: [scriptc](https://github.com/vercel-labs/scriptc) and [Porffor](https://porffor.dev/) compile it ahead of time to native code, and tcm stores it as a content-addressed codebase.

TypeScript as it stands cannot be the language. Its types can lie, it has no way to state failure or capabilities, and it has no value types. So CoreScript is a dialect, neither a strict subset of TypeScript nor a superset, governed by one rule. Wherever TypeScript's types tell the truth, CoreScript computes the same values. Where they would have lied, it rejects the program, traps, or returns a typed absence. It never silently computes a different value. Being a dialect is a real cost, and we judge it necessary for the best experience for both people and agents.

The additions are value types (records, tuples, and structs), enums with payloads and an exhaustive `match`, nominal protocols, checked casts, effect clauses, actors, compile-time evaluation with decorators, and markup. People still read this code, so it has to be easy to read and easy to catch mistakes in. Annotations appear where they state a contract, and a small amount of runtime does the rest.

## Building it

### The toolchain

- The checker is a fork of [tsrs](https://github.com/oxc-project/tsrs), the Oxc project's Rust port of the TypeScript 7 compiler and language server. It ports TypeScript's own scanner, parser, binder, checker, and printer, and it implements the two extension points TypeScript 7.1 added, the compiler API and content mappers. We track it until its stable 7.1 release and then diverge. One obligation remains afterward: the reader for declaration files keeps up with tsc for as long as we consume npm typings, with tsrs's frozen baselines as its regression gate.
- The trees are handled by a fork of [Oxc](https://oxc.rs): its parser, printer, formatter, and transformer. tsrs is built to answer questions about meaning. Parsing new syntax, rendering views, lowering to JavaScript, and rewriting code for refactors are transformation work, and Oxc's arena-allocated tree and in-place traversal are built for that. Oxc also parses the source of npm packages, resolves modules, and minifies the JavaScript we emit, with Rolldown for bundling.
- LLVM generates native code.
- Clang reads C and Objective-C headers, and `swiftc` compiles the bodies that Swift interfaces leave to their clients.

The toolchain is written in Rust.

### Order of work

The first release targets JavaScript, for the web and for servers, and Apple platforms. Android follows, then Windows and Linux. We propose this order:

1. The codebase store, the `core` command over it, and the cold-agent benchmark. Every later decision is measured with them.
2. The dialect on JavaScript: the forked checker, the removals and changed rules, and the oracle against Node. This is the first usable target, for the web and for servers.
3. The additions, each with a specification and conformance tests: effect metadata, values, enums and `match`, actors, and compile-time evaluation.
4. The application architecture and the DOM builder, with rendering on the server. This is the first complete product on one platform.
5. The Apple backend: LLVM, the runtime, the standard library's native floor, and header import, then the UIKit and AppKit builders, then Swift module import and SwiftUI.
6. Source control for many agents, and the registries.
7. Android, with `android.view` and Jetpack Compose, then Windows with WinUI and Linux with GTK.

Two spikes run beside the early steps, because they carry the most technical risk: composing SwiftUI views from code CoreScript generates, and writing Kotlin metadata and Compose's calling convention from a compiler that is not Kotlin's.

## Verification

### Who checks what

A single implementation divides the work of trust. Whether the platforms agree is the compiler's question, answered deterministically. Whether the code meets the requirements is the question that agents and people spend their judgment on.

- The compiler checks types soundly, with every escape hatch visible in a contract and in review. It checks effect clauses on exports, exhaustive `match` and `switch`, sound `readonly`, and that no export reaches a capability its target lacks.
- The same compiler checks every view, on every toolkit, against the shared types. A view that matches over a state and misses a case does not build.
- Tests check behavior against requirements, and their results are evidence about exact hashes.
- People judge intent, and they judge presentation, where nothing is guaranteed to agree across platforms. The goal is to remove the unnecessary duplication that exists today, and separate views are the price of native interfaces.

### Oracles

Every claim of correctness needs something outside the implementation to compare against. For the TypeScript-compatible part of the language, that is Node: erase a CoreScript program the way TypeScript would and run it, and every run in which no check fails must produce the same result. test262 defines the engines' behavior. For the standard library, the oracle is Web Platform Tests and Node's test suite. For a package, it is the package's own tests on each backend. The dialect's additions have no external oracle, and each one needs a specification and conformance tests before it ships.

### The cold-agent benchmark

Many choices in this document are claims about what agents read and write well, and CoreScript starts with no training corpus of its own. Both can be measured. Give an agent that has never seen the language only its documentation and its tools, then compare turns to a passing build and defect rate against TypeScript, Swift, and Kotlin on a fixed set of tasks. The tasks should include editing existing code as well as writing new code, and each run should be repeated with and without the agent-first tooling described above.

Because code is stored as trees, a syntax variant costs a parser and a printer. The benchmark can therefore decide questions that would otherwise be settled by taste: tags with quoted text and plain control flow against JSX with marked control flow, the required `try`, the pipeline operator, and the default isolation of async functions. It can also test the claim behind the multi-toolkit model, by measuring how much faster an agent builds a feature's second view, on a second toolkit, than its first. We should build it early. It has not been run. Until it has, the claim that agents can work in a dialect they were not trained on rests on argument alone.

The benchmark also has to beat a baseline that needs no new language at all: a precise specification plus an oracle, with agents regenerating native code for each platform. If that turns out to be as reliable as one shared implementation, CoreScript is solving a problem that has gone away.

## Open questions

Questions about the language itself are in [DESIGN.md](DESIGN.md#open-questions). The questions below are about the system around it. Each ends with the answer we currently lean toward.

The codebase:

- What does a definition's hash cover? Formatting, comments, and the names of local variables are excluded, and references are replaced by the hashes of what they refer to. Still undecided are whether documentation comments count, how a group of mutually recursive definitions is hashed, how references to npm packages and platform APIs are pinned, and whether the language version is part of the hash. We lean toward hashing a recursive group together, as Unison does, and toward including the language version.
- Where does a nominal type's identity come from? Structs, classes, and actors are nominal, and renaming must stay free, so the name cannot be the identity. We lean toward a unique identifier generated when the type is first added, which is how Unison's unique types work.
- How do agents and people edit without files? The candidates are replacing one definition at a time through a command, editing a scratch buffer that is parsed back into the store, and a materialized directory of `.core` files kept in sync for tools that only understand files. We lean toward the command as the primary path, with the materialized directory for existing editors and agent harnesses.
- Do existing TypeScript 7.1 content mappers, such as those for Vue, Svelte, and MDX, work unchanged? They speak a protocol to an external process and produce TypeScript. We lean toward supporting the protocol for compatibility and preferring mappers written as compile-time code.
- How do the checker's tree and the transformation tree relate? The checker is a port of TypeScript's and is written against TypeScript's tree, and the transformation work uses Oxc's. The stored tree could be converted into the checker's form for each check, both sides could parse text separately and match nodes afterward, or the checker could be rewritten onto Oxc's tree. Oxlint and tsrs already run side by side as separate processes that each parse for themselves. We lean toward one stored tree in Oxc's form, converted to the checker's form one definition at a time, so that new syntax is parsed in one place and tsrs's own parser is left to read declaration files.

The tools:

- What is the command schema written in? It could be a separate schema document. Or the commands could be CoreScript declarations whose signatures are the schema, with `throws` as the error catalog and `uses` saying whether a command changes anything. We lean toward CoreScript declarations.
- Do compiler diagnostics and command errors share one format? We lean toward one envelope with a stable code, a flag saying whether retrying can help, and suggested fixes expressed as edits to definitions.
- When an agent asks about code, does it get data or rendered source? Graph questions, such as what depends on a definition, are data. A contract or a transition table reads better as source. We lean toward read commands that take a view parameter and return rendered source, with JSON for anything that is a list or a graph.

User interfaces and backends:

- How are a feature's per-toolkit views named, and how does a build select among them? We lean toward one name per toolkit under the feature, chosen explicitly by each target's entry point.
- How do async functions and actors map onto each host's concurrency runtime? On Apple platforms the main actor has to be Swift's main actor, since UIKit and SwiftUI are isolated to it. An async function could be a Swift async function at the ABI level, or a CoreScript construct bridged to one. The mapping is not one to one, because a JavaScript promise starts running when it is created and can be awaited later, which a Swift async call cannot. We lean toward building on Swift's concurrency runtime on Apple platforms and on Kotlin's coroutines on Android, as we build on the host's collector.
- What kind of cycle collector does the reference-counted runtime use? Tracing and trial deletion are the candidates, and we have no leaning yet.
- How do records get value equality on the web, where JavaScript compares objects by identity? The compiler can emit an equality function and call it wherever `===` meets a record type, along with hashing and its own `Map` and `Set` for record keys. Or the runtime can intern records, so that equal records are the same object and JavaScript's own `===`, `Map`, and `Set` work unchanged, at the price of a table lookup every time a record is created. We lean toward generated equality, because interface code creates records far more often than it compares them. Under that choice, foreign JavaScript that receives a record still sees identity.

Packages from npm:

- May a package that cannot be compiled natively bring a JavaScript engine with it? Some npm packages use features the dialect removes and are too large to port. On a native target the only way to run such a package is to ship an engine for it. We could allow that as an explicit choice, with the build output naming the package responsible, or refuse, so that such packages are available to web and server code only. We lean toward refusing in the first release.
- Is a value from an unverified declaration file checked or trusted? A declaration file can claim that a function returns a `Config` when the JavaScript behind it returns something else. CoreScript can check the value against the declared type as it enters authored code, which costs time and cannot cover functions. Or it can accept the value and record that the import is trusted without proof, so that reviews and audits can list such imports. We lean toward checking data and recording trust for everything else.

## Alternatives considered

### Kotlin Multiplatform

It shares code across the platforms we care about and has years of production use. The code it generates for JavaScript and Wasm is less than optimal, and Kotlin's defaults hide meaning that an agent has to carry in its head and a reviewer cannot see. Its views are either one Compose interface drawn the same way everywhere or a second language per platform. We also do not want the Kotlin, Gradle, and JVM toolchain as a requirement for every platform.

### A new language

A new language with a Swift-shaped surface would start with typed `throws`, marked call sites, enums with payloads, value types, and argument labels. It would also start with no corpus and no oracle, and it would spend the project's effort on syntax. The features that make Swift attractive are in CoreScript as additions to TypeScript.

### TypeScript as a strict subset

Removing features without adding any would keep every existing tool working unchanged. It leaves no way to state failure or capabilities except by convention, which the next alternative describes.

### Effects by convention

Capabilities could be passed as parameters and errors returned as values, with no new clauses. Each would rewrite every function body or every call, for information the compiler can carry in a clause. Resumable effect handlers in the style of Unison would be more expressive than clauses, and they need continuations, which none of our hosts expose.

### Views in Swift and Kotlin over a shared core

Shared code could end at an exported API, with each platform's views written in Swift, Kotlin, and TypeScript on top of it. That would remove the hardest compiler work in this document. It would also keep three languages, three toolchains, and three sets of idioms in every product, and an agent building one feature would change context once per platform. Writing the views in CoreScript keeps them native and makes the model for building them the same everywhere.

### One interface that runs everywhere

Toolkits such as Flutter and Compose Multiplatform draw one interface on every platform. Projects such as [NSH](https://github.com/longbridge/nsh) pair a Rust core with a rendering shell to the same end, reach the web only as Wasm with about a megabyte of runtime, and ask for a stricter language than consumer applications need. A single interface is the cheapest thing to build. It gives up each platform's conventions and capabilities, which principle 5 exists to keep.

### Emitting Swift and Kotlin source

Transpiling to each host language would make the output look native. Every construct would then have to type-check and mean the same thing in three languages, which makes the smallest of the three the real language. We emit native code and class files.

### An embedded JavaScript engine on mobile

Running shared code in JavaScriptCore, Hermes, or QuickJS would work on the first day and would make every npm package available. It also ships a second runtime and a second heap inside every app, which is the opposite of the goal.

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
- TypeScript 7.1 for the compiler API and content mappers, [tsrs](https://github.com/oxc-project/tsrs) for the checker we fork, and [Oxc](https://oxc.rs) for the tree tools we fork.
- [scriptc](https://github.com/vercel-labs/scriptc) and [Porffor](https://porffor.dev/) for evidence that TypeScript compiles ahead of time to native code, and for testing each program against Node.
- Swift for value semantics, the `as` and `try` families, typed throws, actors, result builders, and the lessons of its macros and of its concurrency defaults.
- React Native and SwiftUI for the promise that one way of building interfaces can be learned once and applied to each platform natively.
- SwiftUI and Jetpack Compose as the declarative toolkits that markup translates into, and the block virtual DOM technique for the imperative ones.
- TC39 proposals for [records and tuples](https://github.com/tc39/proposal-record-tuple), [pattern matching](https://github.com/tc39/proposal-pattern-matching), [first-class protocols](https://github.com/tc39/proposal-first-class-protocols), [do expressions](https://github.com/tc39/proposal-do-expressions), the [pipeline operator](https://github.com/tc39/proposal-pipeline-operator), [decorators](https://github.com/tc39/proposal-decorators), [extractors](https://github.com/tc39/proposal-extractors), and enums.
- TSRX and the [JSX 2.0 discussion](https://github.com/react/jsx/issues/65) for markup.
- The Elm Architecture for the shape of an application.
