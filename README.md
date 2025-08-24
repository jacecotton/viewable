## Viewable
Planning a lightweight class mixin for creating custom elements.

Capitalizing on the arrival of native decorators and some motion on ES signals, DOM parts, and declarative custom elements.

Minimalistic example:
```js
/* click-counter.js */
import {Viewable, state, action, effect} from "viewable";
import view from "./click-counter.view.js";

export default class ClickCounter extends Viewable(HTMLElement) {
  // Observed attributes automatically turned into view-accessible
  // reactive state.
  // Technically mutable by the component, but roughly equivalent
  // to Lit's public "props".
  static observedAttributes = ["label"];

  constructor() {
    super();
    this.attachShadow({mode: "open"});
    // Registers your view function, passes state and actions,
    // schedules first render.
    this.attachView(view);
  }

  connectedCallback() {
    // Post-render imperatives.
  }

  // Additional view-accessible reactive state (non-attribute-reflected).
  // Technically mutable from the outside, but roughly quivalent to Lit's
  // internal "state".
  @state() count = 0;

  // Imperatives associated with the view's inline event listeners and
  // other observers. Responsible for *mutating state*.
  @action() increment() {
    this.count++;
  }

  // Side effects, after-render imperative operations, etc.
  // Responsible for *responding to mutated state* or *deriving state*.
  @effect(["count"]) diagnostics(old) {
    console.log(`this.count updated from ${old.count} to ${this.count}`);
  }
}
```

```js
/* click-counter.view.js */
import {html} from "your-favorite-parser"; // must return TemplateResult

// No `this`. No hooks. No opinions or insight.
export default ({label, count}, {increment}) => html`
  <button @click="${increment}">${label}</button>
  <p>(Button clicked ${count} times)</p>
`;
```

```js
/* click-counter.register.js */
import ClickCounter from "./click-counter.js";
customElements.define("click-counter", ClickCounter);
```

Usage:

```html
<click-counter label="Click me!"></click-counter>
```

Motivation arises from recognizing that, like Lit, custom element imperative logic and behavior, especially side-effectful, should be object-oriented (part of the element specification). But like React, you can eliminate several classes of bugs by having your rendering function be declarative, pure, and ideally, stateless.

Attempting to get the best of both worlds, that is the exact division we're embracing with a small set of semantic decorators for the class-side, and an imperative way to register<sup>1</sup> a view function that automatically and exclusively has access to what it needs (a state snapshot and auto-bound action methods). This results in an even smaller API footprint, and surface area for bugs, than *either* Lit or React, while maintaining relative capability parity—within the following core confines:

1. Eliminating foot-guns only necessary in React or Lit's respective all-or-nothing approaches (React's hook lean, Lit's `this` in templates, etc.)
2. More opinionated philosophy resulting from the enforced separation with seamlessly orchestrated integration.
3. Staying "close to the metal" and foregoing conveniences that are pure sugar—standard for inclusion is only those that provide **both** ergonomic functionality *and* architectural safeguards.
4. Maintaining compatibility and interoperability with multiple renderers so additional production-necessary features (directives, controllers, etc.) can be brought in as needed.

Beyond these guiding principles, we're also aiming to architect this microlibrary in as future-proof a way as possible, influenced in our decisions by the following on-the-horizon APIs:
* DOM Parts (future implementation behind template parser?)
* Signals (future implementation behind `@state` and `@effect`?)
* Declarative Custom Elements

Ideally, swapping out our current implementations with these APIs will be hidden implementation details that should have no effect on the exposed API.

Partly to this end, we are trying to avoid creating a framework, but instead a **progressive enhancement** over the existing ergonomics and conventions of the web standard:
* All decorated class members are still meaningful and operable irrespective of `Viewable` or the decorators.
* The view function can be repurposed and reused by any reactive component system—or even applied naively with `innerHTML`. The `attachView` method just so happens to be our current way of wiring it up.

The highest aspiration of this project is an evolution from a microlibrary to a mere style guide.

----
<small>1. And, in rare cases, invalidate.</small>

### API
#### `attachView(view)`
* `attachShadow` is required

#### `@state(options)`
* transform, equals, should
* custom setters (don't use on getters unless you're memoizing—see [`@memo`](#memo))
* for computed state, probably just do that in the view function? If it's expensive, use memo-state
* How should we handle external subscriptions?
    * We don't want a direct `subscribe` option in the decorator, as it could potentially create duplicate subscriptions and doesn't have a clear way to unsubscribe on disconnect (except for a magical behind-the-scenes cleanup).
    * What we do want is for there to already be an existing subscription to that external store, and then for the decorator to simply specify which key in that store maps to the decorated field (e.g. `@state({store: this.#someRefToStore, selector: (store) => store.currentUser}) currentUser = null`).
    * Then, when that subscription receives an update, the state prop updates and the update pipeline happens. How to implement this, unsure.

#### `@action`

#### `@effect(deps[])`
Returns cleanup method.

##### The future with signals
In the future, signals will provide automatic dependency tracking, at which point `deps[]` will be removed.

So, we shouldn't allow `@effect([])` as an "after-first-render" hook, nor `@effect()` as an "after-every-render" hook.

Instead, for "after-first-render", we should recommend `connectedCallback` (call `super.connectedCallback` either before or after your operations, depending on which you desire). This is more semantically accurate, since after-first-render operations aren't reactive to dependencies, but are instead lifecycle-contingent operations.

We probably shouldn't support an "after-every-render" pattern, as it's likely a code smell.

#### `@memo(deps[])`

#### `invalidateView`

#### `isMounted`

### What about...? (React)
Most divergences from React, and why we can shed so much of its API footprint, derive from the fact that logic and side effects are object-oriented and imperative in a class-based custom element workflow.

* No `useRef`
    * Class fields are non-rendering and persist across renders by default.
    * Get references to rendered nodes with `this.shadowRoot.querySelector`.
* No `useCallback`—class method identity is stable across renders.
* No `useContext`—not needed class-side (just use global state), and for view-side, use Lit's directives to wire up shared context
* No `useLayoutEffect`—`@effect` methods are already synchronous (post-render, pre-paint). Use `setTimeout(cb, 0)` inside `@effect` to do async effects (like `useEffect`).
* No custom hooks—use Lit's directives for reusable stateful logic
* No performance hooks because custom elements leverage the broader browser ecosystem—no `useDeferredValue`, `useTransition`, `useOptimistic` etc.

We provide alternatives or recommend vanilla solutions for the following:

* No `useState`—use `@state` (avoids stale closure issues and surfaces state as regular, r/w-able properties of the element)
* No `useEffect`—use `@effect` for synchronous effects (stable identity, cleaner source of deps, surface effects as regular methods). Use `setTimeout(cb, 0)` inside effect method for async effects.
* No `useMemo`—use `@memo`.
* No `useSyncExternalStore`—use `@state({subscribe})`
* No JSX—use tagged template literals.
* No portals—custom elements use `slot`.

Open questions:

* `@reducer`?

On the horizon:
* Scheduler API can solve for async effects (instead of `setTimeout`).

### What about...? (Lit)
Most divergences from Lit derive from the fact that the lifecycle is handled either more declaratively and semantically (in the case of decorator-based solutions), or idiomatically (where Viewable doesn't have an opinion and doesn't need to provide a hook since doing so would just be sugar).

* No `shouldUpdate`—use [`@state({should: () => {}})`](#state).
* No `willUpdate`
  * Use [`@memo`](#memo-deps) to do things before state updates are considered "complete".
  * Use [`@effect`](#effect-deps) to do things based on previous state values.
  * Use [`@action`](#action) for arbitrary pre-render setup (or `connectedCallback` before calling `super.connectedCallback` for pre-first-render setup).
  * Deliberately no solution provided for "before every render" operations (code smell just like "after every render").
* No `firstUpdated`—first render can be assumed within `connectedCallback` (just make sure to call `super.connectedCallback` first). If you're paranoid, check `this.isMounted`.
* No `updated`—use effects.
* No `createRenderRoot`—shadow DOM required, DIY idiomatically with `attachShadow`.
* No `@query` decorators—use `querySelector`/`querySelectorAll` in `connectedCallback` or `get` method.

We've tweaked other APIs while keeping them effectively the same:

* `invalidateView` instead of `requestUpdate`—Viewable doesn't let you explicitly request an update *per se*. Instead, you declare the view as stale, and the fact that it updates in response is implicit. This is because we don't *really* want you imperatively updating the view and running effects. It should only be done for resynchronizing the view with the environment (observers, events, etc.) This helps preserve a consistent data flow.

Open questions:
* `updateComplete` promise? Potentially useful for outside observers. Also consider dispatching events.

### Known issues
#### Will address
* Typing complexity—how does the view know prop types?
* Testing—modularizing views and decorated fields make testing theoretically easy, but actual utilities need to be developed.
* SSR support—need to research

#### Won't address
* Styles—no opinion (recommendation: constructed stylesheets).
