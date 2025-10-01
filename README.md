Reactivity semantics and views for custom elements, implemented with native decorators (see [tc39/proposal-decorators](https://github.com/tc39/proposal-decorators), [@babel/plugin-proposal-decorators](https://babeljs.io/docs/babel-plugin-proposal-decorators)).

```js
// click-counter.js
import {ViewableComponent, state, action, effect} from "viewable";
import {view} from "./click-counter.view.js";

class ClickCounter extends ViewableComponent(HTMLElement) {
  // Define props as observed attributes.
  static observedAttributes = ["label"];

  // ...or as regular properties decorated with @observable.
  @observable() count = 0;

  constructor() {
    super();
    this.attachShadow({mode: "open"});
    // Attach a view function used as the shadow DOM template.
    this.attachView(view);
  }

  // @action methods are auto-bound and passed to the view to use as
  // handlers for inline events.
  @action() increment() {
    this.count++;
  }

  // @effect methods run in response to prop updates (listed as dependencies).
  @effect(["count"]) debugCount(last) {
    console.debug(`this.count updated from ${last.count} to ${this.count}`);
  }
}

customElements.define("click-counter", ClickCounter);
```
```js
// click-counter.view.js
import {html} from "viewable";

// Props passed as first argument, action methods passed second.
export const view = ({label, count}, {increment}) => {
  return html`
    <button @click="${increment}">${label}</button>
    <p>Clicked ${count} times</p>
  `;
};
```

## Views
Views are stateless (no `this`) and composable DOM templates.

Use tagged template literals (`html`) to break up the template into "parts" that can be individually updated.

### `attachView`
Pass a function reference that returns a view.

`attachView` stores the view function as the shadow DOM's source of truth. When any `observedAttributes` or [`@observable` properties](#observable) change, they (along with [`@action` methods](#action)) are passed to the view function, and then the returned result is applied to the shadow DOM.

## Decorators
### `@observable`
Updates to `@observable` properties rerender the view and rerun corresponding effects, and are usable by both as internal dependencies.
  
If used to create a reactive *property*, **only decorate the `set` method**. This will make the view/effects reactive to property mutation and make the property accessible to the view/effects.

If used in combination with `@computed`, **only define a `get` method**. Using a setter in this case doesn't make sense.

Options:
* `reactive: (previous, next) => false`: The condition under which a state update should trigger an update, based on its previous and next values.
* `coerce: next₁ => next₂`: Transform the value when setting, useful for type coercion (e.g. `{coerce: Number}`) or data normalization (e.g. `{coerce: v => v.trim()}`).
* `equals: (previous, next) => true`: Custom comparator function (what counts as a change). Useful for comparing specific keys within a state prop object (e.g. `(previous, next) => Object.is(previous.id, next.id)`), or using different equality thresholds (e.g., switch to strict comparison via `(previous, next) => previous === next`).
  
#### `@computed()`
Memoized state calculations. Useful, but optional if only relevant to the view function. Only use in combination with `@observable` to decorate `get` methods (in which case, the computed property itself is not reactive—its dependencies are).

### `@action()`
`@action` methods mutate observables in response to the outside world (user events, DOM observers, network responses, etc.) Automatically bound and passed to the view so they can be used as event handlers. Must expect/pass no other arguments except the `event` object.
  
### `@effect()`
`@effect` methods handle any "side effects" of observed changes, running after a listed observable dependency is updated and the resulting re-render has been completed.
  
Method gets `last` object for previous state comparison.

Effects should return callback functions that perform cleanup operations (remove event listeners, observers, timers, subscriptions, etc.) These will be run automatically in `disconnectedCallback`, as well as each time before an effect is invoked.

#### `@effect.once()`
After-first-render instead of after-every-render effects. No dependency array.

## Higher-order concepts
### Context and signals
Shared state:
* Parent &rarr; child: context (no solution yet)
* Global: signals (no integration yet; conceptually easy, keeping an eye on tc39 signals proposal)

### Directives and controllers
Reusable logic:
* Reactive view fragments: directives (use `lit/directive`)
* Reactive class behavior: controllers (no solution yet)

## Contract
### Side effects
#### `observedAttributes` and `attributeChangedCallback`
Attributes listed in `observedAttributes` automatically get reflected as internal properties with camelCased names (`my-attr=""` &rarr; `this.myAttr`), synchronized to the DOM with `get`/`setAttribute`.

Reflected properties are then made reactive through `super.attributeChangedCallback`, which triggers a rerender and reruns effects.

#### `connectedCallback`
`super.connectedCallback` schedules the view's first render.

#### `disconnectedCallback`
`super.disconnectedCallback` runs all effect callbacks for cleanup/teardown, helpful to avoid memory leaks and reverse side effects.

### Constraints
#### Stateless views
Views are meant to be a stateless blueprint (no `this`). Footguns can easily be introduced by reading/writing `window` or `document`, or using browser APIs directly.

#### No reactive lifecycle hooks
Unlike `LitElement`, Viewable does not expose reactive lifecycle hooks.

For the most part, that's what `@effect` (instead of `updated`) or `@effect.once` (instead of `firstUpdated`) are for—atomic, isolated reactions to state changes.

`@observable({equals})` can be used as a "should mutate this property at all" test, while `@observable({reactive})` can be used as a "should cause a re-render and effect re-run" test (instead of `shouldUpdate`).

`@computed` can be used for data preprocessing right before render (instead of `willUpdate`).

#### Requires shadow DOM
No `createRenderRoot`, as by convention, the view is the sole domain of the shadow DOM and vice versa.

## Advantages
### Over `LitElement`
Most potential advantages derive from having [no monolithic lifecycle hooks](#no-reactive-lifecycle-hooks) and encouraging [stateless views](#stateless-views), potentially minimizing the surface area for bugs and making debugging and tracking data flows easier.

### Over React(-like)
Most potential advantages over React and React-like custom element APIs, like Atomico, derive from embracing, isolating, and progressively enhancing the object-oriented approach of the custom elements API, rather than ditching it.

* Since observable values are just upgraded object properties, and the view only ever reads from a state snapshot at render time, no stale closure issues (no `useState`-like set functions necessary).
* Since effects and actions are just upgraded object methods, function identity remains stable across renders by default (unlike `useEffect`; `useCallback` N/A).
* References stored in regular object fields/properties persist across renders (`useRef` N/A).

Keeping views stateless without relying on React's escape hatches maintains many of the advantages of React's functional approach, but without most of the hook fatigue and gotchas.

## Not documented here
* **Styling**—constructable stylesheets are recommended though
* **Template bindings**—refer to `lit-html` documentation
* **SSR support**—Same story as Lit's (see [`lit-labs/ssr`](https://lit.dev/docs/ssr/overview/))
