Reactivity semantics for custom elements, implemented by native decorators (see [tc39/proposal-decorators](https://github.com/tc39/proposal-decorators), [@babel/plugin-proposal-decorators](https://babeljs.io/docs/babel-plugin-proposal-decorators)).

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
import {html} from "viewable";

export const view = ({count, label}, {increment}) => html`
  <button @click="${increment}">${label}</button>
  <p>Clicked ${count} times</p>
`;
```

## Views
A view should be a pure, stateless, composable UI blueprint.

Tagged template literals (with `lit-html/html`) for precise, efficient part-level DOM updates.

### `attachView`
Pass a function that returns an HTML template to use as the component's shadow DOM.

`attachView` automatically passes a snapshot copy of the state at time of rendering, as well as stable references to any action methods. Don't attempt to do so explicitly.
    
### `invalidateView`
Secret. Don't use.

## Decorators
### `@state`
`@state` properties cause re-renders and effect re-runs when updated. Usable by the view and effects as dependencies.
  
If used to create reactive properties, **only decorate the `set` method**, which will both make the property accessible to the view and make the view reactive to its mutation.

If used in combination with `@computed`, **only define a `get` method**. Using a setter in this case doesn't make sense.

`options`:
* `[reactive=(previous, next) => false]`: The condition under which a state update should invalidate the view, based on its previous/old and next/new values.
* `[coerce=next => next]`: Transform the value when setting, useful for type coercion (e.g. `{coerce: Number}`) or data normalization (e.g. `{coerce: v => v.trim()}`).
* `[equals=Object.is(previous, next)]`: Custom comparator function (what counts as a change). Useful for diffing against specific keys within a state prop object (e.g. `(previous, next) => Object.is(previous.id, next.id)`), or using different identity primitives (e.g., switch to strict comparison via `(previous, next) => previous === next`).
  
#### `@computed()`
Memoized state calculations. Useful, but optional if only relevant to the view function. Only use in combination with `@state` to decorate `get` methods (in which case, the computed property itself is not reactive—its dependencies are).

### `@action()`
`@action` methods mutate state in response to the outside world (user events, DOM observers, network responses, etc.) Auto-bound `this` and passed to the view so they can be used as event handlers.
  
### `@effect()`
`@effect` methods handle any "side effects" of state changes, running after a listed state dependency is udpated and the resulting rerender has been completed.
  
Decorated method gets `last` object for previous state snapshot comparison.

Effects should return callback functions that perform cleanup operations (tear down event listeners, observers, timers, subscriptions, etc.) These will be run automatically in `disconnectedCallback`, as well as each time before an effect is invoked.

#### `@effect.once()`
After-first-render instead of after-every-render effects.

## Contract
### Side effects
#### `observedAttributes` and `attributeChangedCallback`
Attributes listed in `observedAttributes` automatically get reflected as internal properties with slugified names (`my-attr=""` &rarr; `this.myAttr`), synchronized to the DOM with `get`/`setAttribute`.

Reflected properties are then made reactive through `super.attributeChangedCallback`, which triggers a rerender and reruns effects.

#### `connectedCallback`
`super.connectedCallback` schedules the view's first render.

#### `disconnectedCallback`
`super.disconnectedCallback` runs all effect callbacks for cleanup/teardown, helpful to avoid memory leaks and reverse side effects.

### Constraints
#### Pure views
Views are meant to be a pure, static blueprint (no `this`). Footguns can easily be reintroduced by reading/writing `window` or `document`, or using browser APIs directly. Instead, mediate through the component class, state assignment, and effects.

#### No reactive lifecycle hooks
Unlike `LitElement`, Viewable does not expose monolithic lifecycle hooks.

For the most part, that's what `@effect` (instead of `updated`) or `@effect.once` (instead of `firstUpdated`) are for—atomic, modularized reactions to state changes.

`@state({equals})` can be used as a "should mutate this property at all" test, while `@state({reactive})` can be used as a "should cause a rerender and effect rerun" test (instead of `shouldUpdate`).

`@computed` is mainly intended to memoize expensive calculations, but also for performing quick "post-mutate, pre-render" calculations (instead of `willUpdate`).

#### Requires shadow DOM
No `createRenderRoot`, as by convention, the view is the sole domain of the shadow DOM, and vice versa.

## Advantages
### Over `LitElement`
Most potential advantages derive from having [no monolithic lifecycle hooks](#no-reactive-lifecycle-hooks) and encouraging [pure views](#pure-views), potentially minimizing the surface area for bugs and making debugging and tracking data flows easier.

### Over Atomico
*(Or React-like custom element libraries in general.)*

Most potential advantages derive from embracing, isolating, and progressively enhancing the object-oriented approach of the custom elements API, rather than ditching it.

* Since state values are just upgraded object properties, and the view only ever reads from a state snapshot at render time, no stale closure issues (no `useState`-like set functions necessary).
* Since effects and actions are just upgraded object methods, method identity remains stable across renders by default (unlike `useEffect`; `useCallback` N/A).
* References stored in regular object fields/properties persist across renders (`useRef` N/A).

Keeping views pure functional without relying on React's escape hatches maintains many of the advantages of React's functional approach, but without most of the hook fatigue and gotchas.

## Roadmap
### Context and signals
To answer React's `useContext` and `lit/context`, considering `@provide` and `@consume`, where "provider" component would share its `@state` collection with the "consumer" component's reactivity system (view and effects).

Also watching [tc39/proposal-signals](https://github.com/tc39/proposal-signals) (stage 1), which not only could power context, but could also be a great solution for truly global, shared state.

It could even be what powers `@state` and `@effect` under the hood. In the meantime, might even adopt a third party signals library, like [`preactjs/signals`](https://github.com/preactjs/signals).

### Alternative renderers
Currently using [`lit-html`](https://lit.dev/docs/libraries/standalone-templates/)'s renderer under the hood, and re-exporting `html` for authoring views. Using it also lets authors steal its [directives](https://lit.dev/docs/templates/directives/) system.

Lit's renderer is probably the best out there for our purposes, but technically could use [µhtml](https://github.com/WebReflection/uhtml) or possibly even JSX.

Keeping an eye on the [DOM Parts proposal](https://github.com/WICG/webcomponents/blob/gh-pages/proposals/DOM-Parts.md), which, depending on the direction it ultimately goes in, might be a replacement for `lit-html`.

### Directives and controllers
Can already use Lit's directives today, which allow for creating "virtual components", or reusing bundles of stateful logic across component views.

Controllers would allow you to reuse bundles of stateful logic and behavior across component *classes*, which is a bit trickier. We have a unique reactivity system, so we can't just lift Lit's Reactive Controllers. Our first task is to solve for namespacing, since views and effects expect flat references for dependencies.

### Reducers
Answer to React's `useReducer()`?

### Typing support
How can views know arg types?

### Testing support
Modularized views and semantic members make testing theoretically easy, but actual utilities need to be developed.

### SSR support
No clue yet, possibly won't solve.

### Debugging and extensibility
Considering littering the `Viewable` class and decorator functions with hooks. This way:
* you could optionally import/enable a debug library that populates `console.debug` messages via the hooks
* you could use the hooks to extend functionality and interoperability with other libraries/APIs
