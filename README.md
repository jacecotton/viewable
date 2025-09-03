## Viewable
Documenting a set of custom element semantics for creating more robust web components, leveraging ES decorators (stage 3 proposal) and coalescing around mature userland conventions.

The vision here is a prescription that downstream implementations, including our own proof-of-concept, can follow.

Minimalistic shape based on current PoC:

```js
// click-counter.js
import {Viewable, state, action, effect} from "viewable";
import {view} from "./click-counter.view.js";

class ClickCounter extends Viewable(HTMLElement) {
  static observedAttributes = ["label"];

  constructor() {
    super();
    this.attachShadow({mode: "open"});
    this.attachView(view);
  }

  @state() count = 0;

  @action() increment() {
    this.count++;
  }

  @effect(["count"]) debugCount(last) {
    console.debug(`this.count updated from ${last.count} to ${this.count}`);
  }
}

customElements.define("click-counter", ClickCounter);
```
```js
// click-counter.view.js
import {html} from "your-favorite-parser";

export const view = ({count, label}, {increment}) => html`
  <button @click="${increment}">${label}</button>
  <p>Clicked ${count} times</p>
`;
```

### Overview
The key elements are (1) [decorators](#decorators) for member semantics ([state](#stateoptions), state-mutating [actions](#action), state-dependent [effects](#effectdeps)), and (2) [stateless views](#view-registration). The idea is to have a clear separation of concerns between the *behavior* side (imperative, object-oriented, stateful, side effectful) and the *view* side (declarative, pure, stateless).

Unlike Lit or React class-based components:
* The view is strictly a pure function of a state snapshot, and is itself forcibly *stateless* (no r/w access to `this`).
* Monolithic lifecycle callbacks are replaced with atomic state signals and effects (decorated fields and methods, respectively).
* Automatic `this`-binding for event handlers (via actions).

And unlike React functional components (or Solid, Atomico, et al.), but like Lit:
* There are no "escape hatches" (hooks) for state management, imperative side effects, and business logic, as those live naturally in the class definition.
* The component view is "always live" (no virtual DOM). Dynamic parts of the rendered DOM are tagged for direct updates (no JSX or algorithmic diffing/reconciliation).
* Method identity and field/property values persist across renders by default, since they're owned by the node object and remain stable until explicit cleanup or garbage collection.

This allows us to shed from both much of the API footprint and resulting surface area for bugs, without losing a minimum set of features and [production hardening](#production-hardening) measures.

### Goal disclaimer
This is not aiming to be a library or framework, but more of a contract for mixins and base classes to implement. The idea here is to isolate functional bits that can be swapped out for competitors or eventually replaced by native features. For example:
* Synchronous view rendering could use `lit/render` (current PoC), `uhtml`, etc., or maybe a native solution ([DOM Parts](https://github.com/tbondwilkinson/dom-parts#readme) might fit the bill depending on shape).
* Asynchronous view rendering can batch behind a custom debouncer, native [microtask](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide) (current PoC solution), or [scheduler](https://developer.mozilla.org/en-US/docs/Web/API/Scheduler) (better but limited availability).
* In our current PoC, generated state `set` methods imperatively request a rerender, and effects are imperatively invoked in response. But you could use proxies or a third-party signals library. In the future, [native signals](https://github.com/tc39/proposal-signals) will be the recommended backend approach.

### Contract
#### Side effects
##### `static observedAttributes[]`
Observed attributes treated as first-class state. Viewable:
* Creates internal properties and syncs to corresponding attributes via `get/setAttribute`.
* Uses `attributeChangedCallback` to trigger rerender on change.
* Passes the internal props to the view function and to the effect runner.

##### `attributeChangedCallback()`
Used as reactive hook for observed attribute changes.

##### `connectedCallback()`
Schedules first render. Move `super.connectedCallback()` around to manage timing of first render relative to your own connect operations.

##### `disconnectedCallback()`
Runs effect callbacks, useful for cleanup to prevent memory leaks, reverse side effects, etc.

#### Internals
##### Render scheduling
Asynchronous (batched, `queueMicrotask` for now, scheduler API later) vs. synchronous (where the `render` invocation lives, updates against `this.shadowRoot` always).

Synchronous function passes a collection of [state field/property values](#state) and [action methods](#action) (with automatic `this` binding) to the registered view function (see [view registration](#view-registration)).

State snapshots are cached until after the next render is complete. This way, the effect runner knows which effects to execute.

#### Public API
##### View registration
* PoC signature: `attachView((state: object, actions: object) => TemplateResult): void`

If a shadow root is attached, registers the passed view function for consumption by the [render scheduler](#render-scheduling).

Replaces Lit/class-based React's `render` lifecycle callbacks.

> <details>
>  <summary>Implementation example</summary>
> 
> ```js
> attachView(viewFn) {
>   if(this.#view || !this.shadowRoot) return;
>   this.#view = viewFn;
> }
> ```
> </details>

> <details>
>  <summary>Usage example</summary>
> 
> ```js
> this.attachView(view);
> ```
> </details>

**❇️ Recommendation:** For `TemplateResult`, `lit-html` offers highly performant part-level diffing and mature [directives](#directives) solution.

##### View invalidation (optional)
* PoC signature: `invalidateView(): void`

A public endpoint for [asynchronous render scheduling](#render-scheduling).

Must not allow passing data through to the view, as this breaks the expected reactive flow.

Replaces Lit's `requestUpdate` utility method, or class-based React's `forceUpdate`.

> <details>
>  <summary>Implementation example</summary>
> 
> ```js
> invalidateView() {
>   this.#renderAsync();
> }
> ```
> </details>

> <details>
>  <summary>Usage example</summary>
> 
> ```js
> this.invalidateView();
> ```
> </details>

> <strong>⚠️ Warning:</strong> Imperative view invalidation should be explicitly discouraged.</summary>
> <details>
> <summary>See details</summary>
> 
> Viewable does not provide a generic `requestUpdate` method (like LitElement does), because views should exclusively be a pure function of state.
>
> The recommended pattern should be to mutate some relevant state property if a view update is needed. Manual invalidation can easily be a footgun because it can be difficult to track down, unlike the `@state`- and `observedAttributes`-based reactivity system.
>
> If a system outside the component instance needs to trigger view updates (e.g. reactive controllers), a relevant state property should be publicly mutable.</details>

#### Decorators
##### `@state({options})`
State fields/properties trigger rerenders when updated, are readable by the view, mutable by actions, and dependable by effects.

If used to create reactive properties, **only decorate the `set` method**, which will both make the property accessible to the view and make the view reactive to its mutation.

If used in combination with `@computed`, **only define a `get` method**. Using a setter in this case doesn't make sense.

`options`:
* `[shouldInvalidate=(previous, next) => true]`: The condition under which a state update should invalidate the view, based on its previous/old and next/new values.
* `[coerce=next => next]`: Transform the value when setting, useful for type coercion (e.g. `{coerce: Number}`) or data normalization (e.g. `{coerce: v => v.trim()}`).
* `[equals=Object.is(previous, next)]`: Custom comparator function (what counts as a change). Useful for diffing against specific keys within a state prop object (e.g. `(previous, next) => Object.is(previous.id, next.id)`), or using different identity primitives (e.g., switch to strict comparison via `(previous, next) => previous === next`).
  
##### `@computed(deps[])`
Memoized state calculations. Useful, but optional if only relevant to the view function. Only use in combination with `@state` to decorate `get` methods (in which case, the computed property itself is not reactive—its dependencies are).

##### `@action()`
Action methods are for mutating state. Auto-bound `this` so they can be used as inline event handlers in the view (all actions passed as an object to the second parameter of the view function).
  
##### `@effect(deps[])`
Effect methods respond to rerenders (side effects, imperative operations, etc.) and are tracked to specific state dependencies.

Decorated method gets `last` object for previous state snapshot comparison.

Effects should return callback functions that perform cleanup operations (tear down event listeners, observers, timers, subscriptions, etc.) These will be run automatically in `disconnectedCallback`, as well as each time before an effect is invoked.

```js
@effect(["delay"]) startPolling() {
  const interval = setInterval(() => this.poll(), this.delay);
  return () => clearInterval(interval);
}
```

Only use for after-every-render. For after-first-render/once-after-mount, use `@effect.once()`.

### Production hardening
There are some things provided by Lit or React(-ish libraries) that Viewable wouldn't care about or provide directly. Instead, we have recommendations.

#### Directives
Useful for reusable state and logic abstractions, aka virtual components.

Virtual components (directives) fundamentally differ from web components (custom elements), in function and purpose. In common between the two is that they're reusable bundles of logic with their own internal state and lifecycle management scheme.

Web components, however, *own* their piece of the UI. Because they have their own shadow roots, their DOM is encapsulated and the styles are scoped. They represent persistent nodes and are thus "hard components".

Virtual components, meanwhile, are useful for either:
* processing and returning data, according to state and lifecycle management specific to them, to be rendered by the host component, or
* as a means to simply "copy-and-paste" DOM snippets that can then be exposed to and manipulated by the host component.

We call them "virtual" because there is no trace in the rendered DOM of the virtual component's identity, only its output.

Implementation notes:
* For PoC, we leverage `lit-html`'s parts API and `lit/directive` to create directives.
* In the future, something like the proposed DOM Parts API may provide an equivalent backend solution, though we will seek to normalize the public API as currently implemented. The view function may end up being a template reference, but registration and passing data should remain ergonomically identical.

#### Styling
Unlike Lit, no direct conveniences for styling. We recommend using constructed stylesheets. Made even easier with CSS module scripts, which are not very publicized but usable.

```js
// Imports as adoptable CSSStyleSheet
import styles from "./my-component.css" with {type: "css"};

class MyComponent extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({mode: "open"});
    this.shadowRoot.adoptedStyleSheets = [styles];
  }
}
```

CSS type assertions aren't well supported, but easily handled by bundlers. Here's a [small Rollup plugin for doing so](https://github.com/jacecotton/rollup-plugin-esm-css-types). Drop when browser support permits.

#### Known issues
##### Will address
* Typing complexity—how does the view know arg types?
* Testability—modularizing views and decorated fields make testing theoretically easy, but actual utilities need to be developed.
* Internal testing—PoC implementation needs robust testing (performance, edge cases, etc.)
* SSR support? May just inherit from Lit. Track conversation for relevant standards proposals.

##### Won't address
* Suspense/skeleton pattern—for now, recommended to render skeletons conditionally based on load state updates. Skeleton components themselves can be part of downstream libs.
