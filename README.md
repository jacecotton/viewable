## Viewable
Planning a lightweight class mixin for creating custom elements.

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
import {html} from "your-favorite-renderer"; // must return TemplateResult

// No `this`!
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

Motivation arises from recognizing that, like Lit, custom element logic and behavior should be imperative, object-oriented, and side-effectful. But like React, you can eliminate several classes of bugs by having your declarative rendering function be pure and stateless.

Attempting to get the best of both worlds, that is the exact division we're embracing with a small set of semantic decorators for the class-side, and an imperative way to register<sup>1</sup> a view function that automatically and exclusively has access to what it needs (a state snapshot and auto-bound action methods). This results in an even smaller API footprint than *either* Lit or React, while maintaining relative feature parity within the following core confines:

1. Eliminating foot-guns only necessary in React or Lit's respective all-or-nothing approaches (React's hook fatigue, Lit's `this` in templates, etc.)
2. More opinionated philosophy resulting from the enforced separation with seamlessly orchestrated integration.
3. Staying "close to the metal" and foregoing conveniences that are pure sugar, including only those that provide **both** ergonomic functionality *and* architectural safeguards.
4. Maintaining compatibility and interoperability with multiple renderers so additional production-necessary features (directives, controllers, etc.) can be brought in as needed.

Beyond these guiding principles, we're also aiming to architect this microlibrary in as future-proof a way as possible, influenced in our decisions by the following on-the-horizon features:
* DOM Parts API
* Signals
* Declarative Custom Elements

Partly to this end, we are trying to avoid creating a framework, but instead a **progressive enhancement** over the existing ergonomics and conventions of the web standard:
* All decorated class members are still meaningful and operable irrespective of what `Viewable` is up to!
* The view function can be repurposed and reused by any reactive component system—or even applied naively with `innerHTML`.

----
<small>1. And, in rare cases, invalidate.</small>
