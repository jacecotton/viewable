## v1.0
The theme will be MVP.

* `ViewableComponent` mixin (extends `ReactiveComponent` base class)
* `observedAttributes` reflection
* `attachView`, `lit-html`, directives
* `@observable`, `@computed`, `@action`, `@effect`, `@effect.once` semantics
* Debug &amp; extension hooks

## v1.1
The theme will be production hardening, performance, and developer experience.

* SSR — can likely lift Lit's directly if we use `lit-html`
* TypeScript migration &amp; tooling
* Testing utilities
* Hooks-based debug library

## v1.2
The theme will be improving abstraction, reusability, composability, etc.

* Controllers (`HeadlessComponent`, extends `ReactiveComponent`)
  * `@effect` patch—respond to controller state
  * Upstream built-in lifecycle callbacks
* Signals migration for `@observable`
* Signals-based implementation of context and global state management

## Maybe one day
* DOM Parts / Declarative Custom Elements synthesis

## To be placed
Not sure when to tackle/how important:
* Cross-boundary idrefs for ARIA
