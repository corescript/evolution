# The Design of the CoreScript Language

> This document describes the CoreScript language as currently envisioned. It accompanies [VISION.md](VISION.md), which explains what the language is for and the system around it. Nothing described here is implemented yet. Each feature will be proposed and reviewed on its own through the CoreScript evolution process, and that review may reject or revise what is sketched here. All syntax shown is illustrative.

## Introduction

CoreScript is a dialect of TypeScript. Its surface is TypeScript's wherever TypeScript already says what we mean. Each addition closes a gap between what a signature states and what the code behind it can do, and each lowers statically, with no runtime protocol and no reflection facility behind it.

Several additions come from TC39 proposals at Stage 1 or 2. Those designs are still moving, so the syntax here is a sketch, and whatever CoreScript adopts becomes its own design from that day.

## One dialect, one rule

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

The same rules apply on every target, the web included. A JavaScript engine would happily run what the dialect removes, but an agent should never have to ask what is allowed where.

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
- `class` is nominal. An instance of one class is never accepted as another because their members happen to match, and an object literal is never an instance. TypeScript treats a class this way only once it has a private member. Interfaces stay structural, and a class satisfies any interface whose members it has. Protocols, described below, are their nominal counterpart.
- `enum` loses reverse lookup, computed members, and declaration merging.
- `namespace` is a name path. A `Button.Props` interface beside a `Button` function, or a class nested under a class, are two definitions at two paths, and nothing merges at run time. Using a namespace as a dynamic bag (`Foo["Bar"]`, `Object.keys(Foo)`) is a compile error.
- `int` is a subtype of `number`. It is closed under `+`, `-`, and `*`, it traps outside the safe-integer range, and division returns `number`.

Declaration files from npm keep tsc's semantics, and authored code gets CoreScript's.

## Values: records, tuples, and structs

JavaScript objects have identity. Two objects with the same fields are different values, any holder of a reference can change the object, and no type says whether a function keeps or mutates what it was given. That is the wrong default for the data an application passes around. State, actions, effects, and messages between actors must be comparable, hashable, and safe to send without asking who else holds a reference.

CoreScript adds value types with the syntax and semantics of TC39's withdrawn Records and Tuples proposal. `#{ x: 1, y: 2 }` is a record and `#[1, 2]` is a tuple. They hold only primitives, enum values, other records and tuples, and references to actors, so a value is immutable all the way down, and `===` compares content. Objects written with `{}`, arrays, and class instances keep the meaning they have in JavaScript.

An actor reference inside a record is compared by identity. A record that holds one can cross between actors like any other. It cannot be serialized, so it cannot cross between client and server or be stored by compile-time code, and the compiler knows which records those are from their types.

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

Declared structs are nominal. `UserId` and `OrderId` are different types even when each holds one string, and the declared name is the one other platforms' languages see. A record type written as a literal, like `Point` above, is structural and exact: a record with an extra field is a different type. Exactness keeps `===` meaning the same thing on every target, and it lets native backends store records inline because their size is known from the type.

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

Structs are meant to hold most of a program's data. Classes are for things that have identity: resources, views, and services.

## Enums and pattern matching

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

`match` is an expression, and it must be exhaustive. Adding a case to `LoadState` stops every `match` that lacks it from compiling, which turns "find every place that handles this" from a search into a list of errors. The `is` operator tests a single pattern. Case patterns such as `LoadState.Loaded(let items)` resolve statically to the enum's declaration.

Two smaller proposals come along because they make expressions compose. A `do { }` expression evaluates to its last expression, so a `match` arm can hold statements. The pipeline operator is pure desugaring that lets standalone functions chain the way methods do.

## Protocols

TypeScript's interfaces are structural: any type with the right members satisfies one, whether or not its author meant it to. That suits the shapes of data that arrive from JavaScript. It does not suit a role that has a meaning, where two types with the same members should not be interchangeable, and it cannot represent the interfaces of other platforms, which are all nominal. CoreScript adds protocols for both.

A protocol is a nominal interface. A type conforms only by saying so, with an `implements` clause, and the compiler checks the conformance where it is declared. The base is TC39's first-class protocols proposal, which is at Stage 1. From it come the `protocol` declaration, `requires` for the members a conforming type must supply, provided members that every conforming type receives, `extends` between protocols, and `implements` on a declaration.

```ts
protocol Ordered {
  requires compare(other: this): Ordering

  lessThan(other: this): boolean {
    return this.compare(other) === Ordering.Less
  }
}

struct Money implements Ordered {
  readonly currency: Currency
  amount: int = 0

  compare(other: Money): Ordering {
    return compareInts(this.amount, other.amount)
  }
}

function max<T extends Ordered>(a: T, b: T): T {
  return a.lessThan(b) ? b : a
}
```

`max` accepts `Money` because `Money` declares the conformance. A struct with an identical `compare` method and no `implements` clause is rejected. Structs, classes, actors, and enums can all conform, and a function that is generic over a protocol dispatches statically.

CoreScript departs from the proposal in the way it does for pattern matching: it keeps the declarations and makes the semantics static. In the proposal, a requirement says only that a property is present, and implementing a protocol copies its members onto an object at run time. Here a requirement is a typed signature, in which TypeScript's `this` type stands for the conforming type, and the compiler resolves conformance with nothing copied. A requirement also carries effect clauses, and a conforming implementation may do less than they allow and never more. A protocol can be isolated to an actor, as SwiftUI's `View` is to the main actor.

Protocols are how other platforms' interfaces appear in CoreScript. Swift and Objective-C protocols, JVM interfaces, WinRT interfaces, and GObject interfaces are all nominal, and each is imported as a protocol. In the other direction, a CoreScript protocol is emitted as the platform's own kind: a Swift protocol with its witness tables, or a JVM interface. A CoreScript type that implements SwiftUI's `View` is therefore a view as far as SwiftUI can tell. Interfaces from TypeScript declaration files stay structural.

We expect the guarantees the compiler already tracks about values to become protocols that it derives: equality, hashing, the ability to cross between actors, and the ability to be serialized. A generic function can then require any of them by name.

## Checked casts and traps

TypeScript's `as` is an unchecked assertion. `value as User` compiles whether or not the value is a `User`, and nothing happens at run time. CoreScript splits the operator in three, parallel to Swift's and to the `try` family.

- `as` keeps TypeScript's meaning wherever that meaning is safe. Every downcast that tsc accepts becomes a compile error that points at the two checked forms.
- `as?` checks at run time and evaluates to `T | undefined`.
- `as!` checks at run time and traps if the check fails.

The target of a checked cast has to be something the runtime can test, so function types and bare type parameters are rejected, and the target of `as?` cannot include `undefined`, since failure would be indistinguishable from success. `as const` and `satisfies` are unchanged. Both new spellings are errors in TypeScript today, so no existing program changes meaning.

A trap is a bug in the program, and it sits deliberately outside the error system. A failed `as!`, a failed `!`, and `int` overflow all trap. Traps never appear in a `throws` clause, and `catch` does not catch them on any target.

## Effect clauses

Three clauses state what a function's body can do: `throws(E)`, `uses(C)`, and `isolated(A)`. [VISION.md](VISION.md#effect-metadata) describes them and what the system does with them. They are inferred for every definition, and they are required and checked on exports. A call that can throw is marked with `try`, `try?`, or `try!`, parallel to the three casts.

## Actors and isolation

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

Only immutable data crosses between actors: records, tuples, primitives, enum values, and references to actors. Passing a mutable object or a closure across is a compile error. Values make message passing cheap, and they make the rule simple enough to check from types alone. The same rule covers the boundary between client and server.

There is a predefined main actor. Top-level mutable bindings belong to it, which gives ordinary single-threaded code the meaning it has today and makes any other actor `await` to touch them. Platform toolkits that must be called on the main thread, such as UIKit and Android's views, are imported as isolated to it.

Isolation is the third kind of effect metadata. A function that synchronously touches an actor's state is inferred `isolated` to that actor, so the annotation Swift programmers write by hand is mostly computed here. The clause names an actor, never an executor: `isolated(Main)`, `isolated(account)` for a parameter, and an implicit `isolated(this)` on an actor's methods. A function holds at most one isolation. Touching two actors synchronously in one body is a compile error that points at `await`.

Where a function runs is readable from its signature:

- A synchronous function runs in its caller's domain. It cannot suspend, so it cannot move.
- A method on a class runs where the instance lives, and a closure runs where it was created, because neither can cross.
- An async function with an `isolated` clause runs on that actor.
- A free async function with no clause runs in no actor's domain. Natively that is the pool.
- An async function that takes mutable arguments or closures says `isolated(caller)`, which is never inferred.

The fourth rule is Swift's original one, from [SE-0338](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0338-clarify-execution-non-actor-async.md). Swift 6.2 moved away from it in [SE-0461](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md), so that async functions run on the caller's actor by default. We keep the original on purpose. Under it, a mistake is a compile error at the call, and the error names the fix. Under the newer default, the mistake is heavy work running on the main actor, which no checker reports. Swift changed in part because the old default made async functions that take non-sendable values unusable from actors, and `isolated(caller)` covers that case explicitly. We expect the [benchmark](VISION.md#the-cold-agent-benchmark) to confirm this choice or overturn it.

## Compile-time evaluation and decorators

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

## Markup

CoreScript's markup builds on JSX's tags and changes how they compose. Inside a builder block, written `@{ }`, every statement contributes to the result. Text is an ordinary quoted string, and `if` and `for` are the language's own, with no template-only variants. Two cleanups ride along from the JSX 2.0 discussions: HTML entities are not decoded, and attributes have a shorthand.

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

The semantics come from Swift's result builders. The type a block is expected to have selects a builder, the tags resolve to the element types of that builder's toolkit, and the block lowers to static calls. Builders are ordinary libraries. [VISION.md](VISION.md#multi-toolkit-user-interfaces) lists the toolkits and describes the two ways a block reaches them.

For an imperative toolkit, the lowering passes on what the compiler knew about the block's structure. Plain JSX calls one factory per element and hands the library a fresh tree of generic nodes on every render. A builder lowering looks like this:

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

Each template is a compile-time constant with a content hash, and a render is template identities plus the values in their holes. Comparing two renders compares a few values, which is the block virtual DOM technique, and the library never has to rediscover that a heading is static or that a branch has two arms. View identity falls out of the same structure: a template at a slot path is one view, the other arm of an `either` is a different one, and each row of an `each` is identified by its `key`.

For a declarative toolkit, the same block lowers to that toolkit's own constructs: SwiftUI's generic view types, or composable calls for Compose. The toolkit's runtime then provides identity and diffing.

Keys stay explicit. Records have equality and hashing from the compiler, so `key item` is always available, and it is right for items with no lifecycle, such as tags. An entity that changes over time needs a stable identifier, or editing its title would destroy and recreate its row.

## Deliberately left out

- Ownership, borrowing, and lifetime annotations. Reference counting with a cycle collector, or the host's garbage collector, manages memory.
- Compatibility with TypeScript as it evolves. After the fork point, new JavaScript or TypeScript features are adopted only when we want them and must each go through the CoreScript evolution proposal process individually.
- A `const` that makes class instances deeply immutable. A constant binding cannot stop writes through another alias, copying on assignment would silently change what existing TypeScript code computes, and foreign typings do not say which methods mutate their receiver.
- User-defined matchers. TC39's pattern matching proposal builds them on a `Symbol.customMatcher` protocol looked up at run time, and CoreScript's patterns resolve statically.
- TC39's structs and shared structs. Classes already have a fixed layout, and shared structs permit data races by design.
- Compile-time code that produces types. Generating declarations belongs to deriving decorators.
- The imperative half of TC39's protocols proposal: `Protocol.implement()`, the `Protocol()` constructor, `Protocol.describe()`, and `Protocol.union()`. Conformance is declared and resolved statically.

[VISION.md](VISION.md#alternatives-considered) covers the alternatives to the larger choices, such as capabilities passed as parameters.

## Future directions

- Named global actors beyond the main actor.
- Library APIs from TC39 proposals that need no new syntax, adopted one proposal at a time.

## Open questions

Each question ends with the answer we currently lean toward. Questions about the system around the language are in [VISION.md](VISION.md#open-questions).

Error handling:

- How is a block that handles errors written? With `try` as the marker on each throwing call, JavaScript's statement reads `try { const cart = try loadCart(id) } catch (e) { }`. Swift has the same marker and spells the block `do { } catch { }`. CoreScript already has `do { }` as an expression, so a `do` with a `catch` clause would be that construct with a handler attached, and it would yield a value. The parser can tell it from `do { } while` by the keyword after the block. We lean toward `do { } catch { }`, with JavaScript's `try { }` block rejected by a diagnostic that names the replacement, and toward letting the benchmark overrule that.

Absence:

- What is the policy for `null` and `undefined`? JavaScript has two values for absence, and Swift and Kotlin have one each. The language's own constructs produce `undefined`: `as?`, `try?`, optional fields, and missing arguments. `null` arrives from JSON and from the DOM. The dialect rule means neither can change meaning in existing code. Undecided are whether exported signatures may use `null`, how a type that admits both crosses to a platform with one absent value, and what each means when a record is serialized. We lean toward `undefined` in exported signatures, `null` only in types that originate in foreign declarations, both mapping to the platform's single absent value, and a compile error for an export whose type admits both.

Enum payloads:

- Are a payload's fields named, positional, or either? We lean toward a case declared like a parameter list, constructed like a call, and matched by position, with the names becoming labels on other platforms as parameter names do.
- May an enum declare methods, getters, and statics, as a struct can? We lean toward yes.
- May one enum mix cases that carry payloads with cases that have numeric or string values? We lean toward no.
- May a payload hold a class instance? We lean toward the rule for records, so that every enum value is comparable and can cross between actors.
- May an enum be generic, as in `Option<T>`, and recursive? We lean toward yes to both, with the compiler boxing a recursive payload.

The pipeline operator:

- Does it ship in the first version? It is the least necessary addition. We lean toward measuring it in the benchmark first.
- If it ships, which token names the piped value? TC39's placeholder `%` reads as the remainder operator, and `#` and `@` already begin records, tuples, builder blocks, and decorators here. We have no leaning yet.

Deriving decorators:

- Is a deriving decorator applied with the same `@name` syntax as a wrapping one? We lean toward yes, with the decorator's own signature saying which kind it is.
- Where do generated declarations live? We lean toward members under the target's name path only, so a decorator on `Cart` can add `Cart.lenses` and cannot add an unrelated top-level name.
- May handwritten code refer to generated names? We lean toward yes from function bodies and no from the signatures a decorator reads, since expansion runs after declared signatures are checked.
- May a deriving decorator be applied to a declaration its author does not own, such as a type from a library? We lean toward no at first.
- Can generated definitions be edited? We lean toward no. They are stored by hash and viewable, and they are regenerated when their inputs change.

Protocols:

- Are a protocol's members plain names or symbols scoped to the protocol? TC39's proposal makes them symbols by default, written `value[Ordered.compare](other)`, so that two protocols can never collide, and offers plain names as an option. Swift, Objective-C, and the JVM all use plain names. We lean toward plain names, with a qualified form for the rare collision.
- How are associated types and static requirements written? Swift's protocols have both, and SwiftUI's `View` uses both. TC39's proposal has neither, having removed static members in February 2026. Both are required here, and we have no leaning on their syntax.
- How are Objective-C's optional requirements represented? UIKit's delegate protocols consist almost entirely of them. We lean toward an optional marker on a requirement, allowed only in imported protocols.
- How does a type declared elsewhere gain a conformance? TC39's proposal does it with a call at run time. We lean toward a standalone conformance declaration, stored as a definition of its own, permitted only when the protocol or the type belongs to the codebase declaring it, with one conformance per pair in a program.
- How is a protocol used as the type of a value? As a generic constraint it is resolved statically. As the type of a parameter or a result it can mean one concrete type that the caller cannot see, or a box that holds any conforming value. Swift spells these `some` and `any`. We lean toward adopting both spellings and rejecting a bare protocol name in that position.
- Can conformance be tested at run time, with `as?` or with the proposal's `implements` operator? We lean toward allowing it for class instances, actor references, and boxed values, which carry their type at run time, and rejecting it for records.
- Which built-in guarantees become protocols? We lean toward equality, hashing, crossing between actors, and serialization, all derived by the compiler and never implemented by hand.

Markup:

- How are Swift's unlabeled arguments, argument labels, modifiers, and trailing closures written as attributes and children? Compose passes its modifiers as an ordinary argument, so the hard case is SwiftUI, where the order of modifiers matters. This needs prototypes, and we have no leaning yet.
- What form does a view declaration take? The examples in [VISION.md](VISION.md#one-model-for-views) use a function from state to a builder block. A view could be a struct, as in SwiftUI. It could be a function that renders on every update, as in React, one that runs once to set up, as in Solid, or a setup function that returns a render function, as in Remix 3. It could also be any of a function, an async function, a generator, or an async generator, as in [Crank](https://crank.js.org/), where a generator's local variables are the view's own state. The choice decides where a view keeps its transient state. SwiftUI's unit is a struct and Compose's is a function, so the compiler translates on one side whichever form we choose. A struct view can hold a state value and a reference to the store's actor, and cannot hold a closure. The proposal process will settle this, and we have no leaning yet.

The checker:

- How does one checker apply two sets of rules? Declaration files from npm keep tsc's semantics, and authored code gets CoreScript's. A rule about a statement is easy to apply by origin. A rule that lives in assignability is harder, because the relation compares two types and caches the answer, and the types may come from either side. Sound `readonly`, nominal classes, and exact records are such rules. Conditional types inside npm typings also evaluate assignability, so a stricter relation could change what those typings compute. We lean toward choosing the rule by where a check is written: authored code is always checked strictly, and a conditional type is evaluated under the rules of the file that declared it, with separate caches.
