## Viewable
Viewable is an alternative to Lit's reactivity engine (`LitElement`) while still leveraging its rendering engine (`lit-html`) under the hood.

It provides a way to register a [view](#views) as a UI template, and then a small set of member semantics (via decorators) to create and handle reactivity.

```js
// click-counter.js
import {Viewable, state, action, effect} from "viewable";
import {view} from "./click-counter.view.js";

class ClickCounter extends Viewable(HTMLElement) {
  static observedAttributes = ["label"];

  @state() count = 0;

  constructor() {
    super();
    this.attachShadow({mode: "open"});
    this.attachView(view);
  }

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
import {html} from "lit-html";

export const view = ({count, label}, {increment}) => html`
  <button @click="${increment}">${label}</button>
  <p>Clicked ${count} times</p>
`;
```

### Definitions
#### Views
**Views** are the HTML template used to render the UI of a component.

Importantly, views only have access to:
1. a **snapshot copy** of [state](#state) at time of rendering (hence views are "stateless" or "pure functions of state"—no access to `this`), and
1. **stable references** to a class's [actions](#actions), which are methods that react to the *outside world* (such as a user clicking a button) by changing the corresponding state of a component.

This way, the view strictly reflects relevant state at the moment it was asked to render, and cannot produce its own instance-level side effects.<sup>1</sup> Views are thus the declarative and pure counterpart to [effects](#effects).

<sup>1. It can still technically produce *general* side effects and impure results through access to `window`, `document`, and browser APIs, but this is more of a linting concern than an architectural one.</sup>

#### State
**State** are fields and properties that the [view](#views) and [effects](#effects) automatically react to and use as internal dependencies.

The view has access to a copy of the state at render time, and automatically rerenders when it changes.

Effect methods are impure functions of state (live state access, imperatives/side effects), and will run when state listed as a dependency is changed.

A custom element's observed attributes are automatically treated as state.

#### Actions
Actions are methods that react to the *outside world* (user events, external state updates, etc.) by changing the component's state.

Actions are automatically bound to `this` and then passed by reference to the view. Their identity is stable across renders and are thus usable as handlers for inline events.

#### Effects
Effects perform imperative operations (like setting up or reconfiguring timers, observers, DOM queries, etc.) in response to state changes. Unlike the view, effects live in the class and have access to live state, as they *are* intended for side effects.

Effects should return callback functions used for teardown/cleanup. These callbacks will always be invoked before the effect itself reruns, as well as in `disconnectedCallback`. This can help prevent memory leaks and reverse side effects.

### Core principles
* **Separation of concerns** — Class = behavior and imperative operations, view = UI description.
* **Progressive enhancement** — State, actions, and effects are each just regular class members, upgraded by decorators. The view is just a UI blueprint that doesn't care about how it's rendered or from where it gets its data.
* **High level abstraction** — No additional, monolithic lifecycle callbacks, and no hook fatigue from complicated mental maps. No virtual DOM wrinkles to trip on.

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

### Internals
#### Render pipeline
Asynchronous (batched, `queueMicrotask` for now, scheduler API later) vs. synchronous (where the `render` invocation lives, updates against `this.shadowRoot` always).

Synchronous function passes a collection of [state field/property values](#state) and [action methods](#action) (with automatic `this` binding) to the registered view function (see [view registration](#view-registration)).

State snapshots are cached until after the next render is complete. This way, the effect runner knows which effects to execute.

#### Collections

### Public API
#### View registration
If a shadow root is attached, registers the passed view function for consumption by the [render scheduler](#render-scheduling).

Replaces Lit/class-based React's `render` lifecycle callbacks.

#### View invalidation
A public endpoint for [asynchronous render scheduling](#render-scheduling).

Must not allow passing data through to the view, as this breaks the expected reactive flow.

Replaces Lit's `requestUpdate` utility method, or class-based React's `forceUpdate`.

> <strong>⚠️ Warning:</strong> Imperative view invalidation is explicitly discouraged.</summary>
> <details>
> <summary>See details</summary>
> 
> Viewable does not provide a generic `requestUpdate` method (like `LitElement` does), because views should exclusively be a pure function of state.
>
> The recommended pattern should be to mutate some relevant state property if a view update is needed. Manual invalidation can easily be a footgun because it can be difficult to track down, unlike the `@state`- and `observedAttributes`-based reactivity system.
>
> If a system outside the component instance needs to trigger view updates (e.g. reactive controllers), a relevant state property should be publicly mutable.</details>

#### `@state()`
Bundled in `this[STATE_COLL]`.

If used to create reactive properties, **only decorate the `set` method**, which will both make the property accessible to the view and make the view reactive to its mutation.

If used in combination with `@computed`, **only define a `get` method**. Using a setter in this case doesn't make sense.

`options`:
* `[shouldInvalidate=(previous, next) => true]`: The condition under which a state update should invalidate the view, based on its previous/old and next/new values.
* `[coerce=next => next]`: Transform the value when setting, useful for type coercion (e.g. `{coerce: Number}`) or data normalization (e.g. `{coerce: v => v.trim()}`).
* `[equals=Object.is(previous, next)]`: Custom comparator function (what counts as a change). Useful for diffing against specific keys within a state prop object (e.g. `(previous, next) => Object.is(previous.id, next.id)`), or using different identity primitives (e.g., switch to strict comparison via `(previous, next) => previous === next`).
  
#### `@computed()`
Memoized state calculations. Useful, but optional if only relevant to the view function. Only use in combination with `@state` to decorate `get` methods (in which case, the computed property itself is not reactive—its dependencies are).

#### `@action()`
Bundled in `this[ACTIONS_COLL]`
  
#### `@effect()`
Bundled in `this[EFFECTS_COLL]`

Decorated method gets `last` object for previous state snapshot comparison.

Effects should return callback functions that perform cleanup operations (tear down event listeners, observers, timers, subscriptions, etc.) These will be run automatically in `disconnectedCallback`, as well as each time before an effect is invoked.

##### `@effect.once()`
After-first-render instead of after-every-render effects.

### Roadmap
In a future phase, will work on:
* Context — `@provide` and `@consume` for shared context across custom elements (shares provider's `@state` collections with consumer's reactivity system).
* Controllers — Can use directives (reusable bundles of state and lifecycle across views) as-is with `lit/directive`, but controllers are trickier (reusable bundles of state and logic across classes). Need to solve for namespacing given that views and effects expect flat references.
* Typing
    * External — Need to solve for how the view can know arg types.
    * Internal — Probably look into migrating to TS.
* Testing
    * External — Modularized views and decorated members make testing theoretically easy, but actual utilities need to be developed.
    * Internal — Need our own tests for Viewable development.
* SSR — No clue yet, probably won't solve.
* What does the picture look like for private state and actions when used as view or effect dependencies?
