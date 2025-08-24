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

## `attachView(view)`
* `attachShadow` is required

## `@state`
* transform, equals, should
* custom setters (don't use on getters unless you're memoizing—see [`@memo`](#memo))

## `@action`

## `@effect(deps[])`

## `@memo(deps[])`

## `invalidateView`

## `isMounted`

## What about...? (React)
Most divergences from React, and why we can shed so much of its API footprint, derive from the fact that logic and side effects are object-oriented and imperative in a class-based custom element workflow.

* No `useRef`
    * Class fields are non-rendering and persist across renders by default.
    * Get references to rendered nodes with `this.shadowRoot.querySelector`.
* No `useCallback`—class method identity is stable across renders.
* No `useContext`—use Lit's directives.
* No custom hooks—use Lit's directives.
* No `useMemo`—use `@memo`.
* No JSX—use tagged template literals.

## What about...? (Lit)
Most divergences from Lit derive from the fact that the lifecycle is handled either more declaratively and semantically (in the case of decorator-based solutions), or idiomatically (where Viewable doesn't have an opinion and doesn't need to provide a hook since doing so would just be sugar).

* No `shouldUpdate`—use [`@state({should: () => {}})`](#state).
* No `willUpdate`
  * Use [`@memo`](#memo-deps) to do things before state updates are considered "complete".
  * Use [`@effect`](#effect-deps) to do things based on previous state values.
  * Use [`@action`](#action) for arbitrary pre-render setup (or `connectedCallback` before calling `super.connectedCallback` for pre-first-render setup).
  * Deliberately no solution provided for "before every render" operations (code smell).
* No `firstUpdated`—first render can be assumed within `connectedCallback` (just make sure to call `super.connectedCallback` first). If you're paranoid, check `this.isMounted`.
* No `updated`—use effects.

## Known and open issues
* Typing complexity—how does the view know prop types?
* Testing—modularizing views and decorated fields make testing theoretically easy, but actual utilities need to be developed.
