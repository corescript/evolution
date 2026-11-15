# CoreScript

CoreScript is a programming language for building a whole product in one language, on every platform it ships on. It is a dialect of TypeScript, designed for agents to write and for people to review. Everything except a product's views is written once, and each platform's views are written in the same language against that platform's own toolkit.

Nothing here is implemented yet. This repository holds the design.

## What defines it

- A content-addressable codebase. Code is stored as definitions keyed by hash, and text is a view.
- Effect metadata. Every definition carries what it can throw, which capabilities it uses, and where it must run.
- Distributed package registries on the AT Protocol.
- Source control, a command line, and language intelligence built for agents.
- Multi-target compilation: JavaScript for the web and for servers, native code that follows Swift's ABI on Apple platforms, and class files for Android.
- Multi-toolkit user interfaces: one declarative model for views, written once per toolkit, from the DOM to SwiftUI and Jetpack Compose.

## A taste

An exported function states how it can fail and what it needs, and every call that can throw is marked. The syntax is illustrative.

```ts
export async function loadCart(id: CartId): Promise<Cart>
  throws(NotFound | NetworkError)
  uses(network)
{
  const response = try await fetch(cartUrl(id))
  return try parseCart(await response.text())
}
```

## Documents

- [VISION.md](VISION.md) explains what CoreScript is for, states the eight principles the design follows, and describes the system around the language.
- [DESIGN.md](DESIGN.md) describes the language itself.
- [docs/](docs/) holds background: the application architecture, a guide to command lines built for agents, and a brief on what agents can build today.

## Status

The project is at the design stage. The first release targets JavaScript and Apple platforms, and [VISION.md](VISION.md#order-of-work) proposes the order of work.
