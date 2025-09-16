Reactivity semantics and views for custom elements, implemented with native decorators (see [tc39/proposal-decorators](https://github.com/tc39/proposal-decorators), [@babel/plugin-proposal-decorators](https://babeljs.io/docs/babel-plugin-proposal-decorators)).

```js
// click-counter.js
import {ViewableComponent, state, action, effect} from "viewable";
import {view} from "./click-counter.view.js";

class ClickCounter extends ViewableComponent {
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

export const view = (state, actions) => {
  const {label, count} = state;
  const {increment} = actions;

  return html`
    <button @click="${increment}">${label}</button>
    <p>Clicked ${count} times</p>
  `;
};
```

## Views
Views are stateless (no `this`) and composable UI blueprints.

Use tagged template literals (`html`) to break up the template into "parts" that can be individually updated.

### `attachView`
Pass a function that returns a view to use as the component's shadow DOM.

`attachView` automatically passes a snapshot of [`@state` properties](#state) at render time, as well as bound references to any [`@action` methods](#action).

## Decorators
### `@state`
Updates to `@state` properties re-render the view and re-run corresponding effects, and are usable by both as internal dependencies.
  
If used to create a reactive *property*, **only decorate the `set` method**. This will make the view/effects reactive to property mutation and make the property accessible to the view/effects.

If used in combination with `@computed`, **only define a `get` method**. Using a setter in this case doesn't make sense.

Options:
* `reactive: (previous, next) => false`: The condition under which a state update should trigger an update, based on its previous and next values.
* `coerce: next₁ => next₂`: Transform the value when setting, useful for type coercion (e.g. `{coerce: Number}`) or data normalization (e.g. `{coerce: v => v.trim()}`).
* `equals: (previous, next) => true`: Custom comparator function (what counts as a change). Useful for diffing against specific keys within a state prop object (e.g. `(previous, next) => Object.is(previous.id, next.id)`), or using different identity primitives (e.g., switch to strict comparison via `(previous, next) => previous === next`).
  
#### `@computed()`
Memoized state calculations. Useful, but optional if only relevant to the view function. Only use in combination with `@state` to decorate `get` methods (in which case, the computed property itself is not reactive—its dependencies are).

### `@action()`
`@action` methods mutate state in response to the outside world (user events, DOM observers, network responses, etc.) Automatically bound and passed to the view so they can be used as event handlers. Must expect/pass no other arguments except the `event` object.
  
### `@effect()`
`@effect` methods handle any "side effects" of state changes, running after a listed state dependency is updated and the resulting re-render has been completed.
  
Method gets `last` object for previous state comparison.

Effects should return callback functions that perform cleanup operations (remove event listeners, observers, timers, subscriptions, etc.) These will be run automatically in `disconnectedCallback`, as well as each time before an effect is invoked.

#### `@effect.once()`
After-first-render instead of after-every-render effects. No dependency array.

## Higher-order concepts
### Context and signals
Shared state:
* Parent &rarr; child: context (no solution yet)
* Global: signals (no integration yet; conceptually easy, keeping an eye on ts39 signals proposal)

### Directives and controllers
Reusable logic:
* Reactive view fragments: directives (use `lit/directive`)
* Reactive class behavior: controllers (no solution yet)

#### Controller exploration
Under consideration is a `ViewableComponent` and then a `HeadlessComponent`. Both would extend a `ReactiveComponent` class that handles most internals—state, actions, effects, lifecycle—but only a `ViewableComponent` would be able to attach and render a view. A `HeadlessComponent` could then either stand alone, or be able to be registered as a controller for a `ViewableComponent`.

The way this would work is that you would namespace the headless component during registration, and then all of the dependencies upstreamed from the controller would be accessible to the host component.

```js
// ViewableComponent extends ReactiveComponent with view internals and lifecycle.
class ComponentA extends ViewableComponent {
  @state() baz = "qux";

  // ComponentB is instanced by ComponentA instantiation.
  @controller(ComponentB) b;

  constructor() {
    super();
    // Since attachShadow is a requirement of attachView, we should just
    // abstract it. A "view" implies a shadow root.
    this.attachView(view);
  }

  @effect(() => ["baz", this.b.foo]) respond() {
    console.log(`Component A sees Component A baz`, this.baz);
    console.log(`Component A sees Component B foo`, this.b.foo);
  }
}

// HeadlessComponent runs ReactiveComponent in headless mode.
class ComponentB extends HeadlessComponent {
  @state() foo = "bar";

  @effect.once() mount() {
    this.foo = "baz";
  }
}
```

This would come with significant considerations and leaves many open questions:
* Effect deps can now accept a function that returns an array and is then bound by the decorator. This would require some sort of hash-matching against the dependency collection generated at initialization, which could get weird. The alternative is to keep a flat array, but namespace the dependencies (e.g. `["baz", "b:foo"]`), but this seems more brittle and less idiomatic/more magical.
* Might need to consider adding phase control for effects (`"layout" | "passive" | "postrender"`).
* Does `ComponentA` ever run `ComponentB`'s effects? Especially, for example, `@effect.once` methods?
* Debugging is going to be especially important—dependency graphs, data flows, etc. could get complicated depending on implementation and downstream usage.
* `HeadlessComponent` needs to be able to hook into the lifecycle of the host component because that's what a *controller* is—it's basically able to manipulate any part of the host component but as a safe, namespaced "shadow".

## Contract
### Side effects
#### `observedAttributes` and `attributeChangedCallback`
Attributes listed in `observedAttributes` automatically get reflected as internal properties with camelCased names (`my-attr=""` &rarr; `this.myAttr`), synchronized to the DOM with `get`/`setAttribute`.

Reflected properties are then made reactive through `super.attributeChangedCallback`, which triggers a re-render and re-runs effects.

#### `connectedCallback`
`super.connectedCallback` schedules the view's first render.

#### `disconnectedCallback`
`super.disconnectedCallback` runs all effect callbacks for cleanup/teardown, helpful to avoid memory leaks and reverse side effects.

### Constraints
#### Stateless views
Views are meant to be a stateless blueprint (no `this`). Footguns can easily be introduced by reading/writing `window` or `document`, or using browser APIs directly, so don't do that.

#### No reactive lifecycle hooks
Unlike `LitElement`, Viewable does not expose monolithic lifecycle hooks.

For the most part, that's what `@effect` (instead of `updated`) or `@effect.once` (instead of `firstUpdated`) are for—atomic, modularized reactions to state changes.

`@state({equals})` can be used as a "should mutate this property at all" test, while `@state({reactive})` can be used as a "should cause a re-render and effect re-run" test (instead of `shouldUpdate`).

`@computed` is mainly intended to memoize expensive calculations, but also for performing quick "post-mutate, pre-render" calculations (instead of `willUpdate`).

#### Requires shadow DOM
No `createRenderRoot`, as by convention, the view is the sole domain of the shadow DOM and vice versa.

## Advantages
### Over `LitElement`
Most potential advantages derive from having [no monolithic lifecycle hooks](#no-reactive-lifecycle-hooks) and encouraging [stateless views](#stateless-views), potentially minimizing the surface area for bugs and making debugging and tracking data flows easier.

### Over Atomico
Most potential advantages derive from embracing, isolating, and progressively enhancing the object-oriented approach of the custom elements API, rather than ditching it.

* Since state values are just upgraded object properties, and the view only ever reads from a state snapshot at render time, no stale closure issues (no `useState`-like set functions necessary).
* Since effects and actions are just upgraded object methods, function identity remains stable across renders by default (unlike `useEffect`; `useCallback` N/A).
* References stored in regular object fields/properties persist across renders (`useRef` N/A).

Keeping views stateless without relying on React's escape hatches maintains many of the advantages of React's functional approach, but without most of the hook fatigue and gotchas.

## Roadmap
### Alternative renderers
Currently using [`lit-html`](https://lit.dev/docs/libraries/standalone-templates/)'s renderer under the hood, and re-exporting `html` for authoring views. Using it also lets authors steal its [directives](https://lit.dev/docs/templates/directives/) system.

Lit's renderer is probably the best out there for our purposes, but technically could use [µhtml](https://github.com/WebReflection/uhtml) or possibly even JSX.

Keeping an eye on the [DOM Parts proposal](https://github.com/WICG/webcomponents/blob/gh-pages/proposals/DOM-Parts.md), which, depending on the direction it ultimately goes in, might be a replacement for `lit-html`.

### Typing support
How can views know state property and action return types?

### Testing support
Modularized views and semantic members make testing theoretically easy, but actual utilities need to be developed.

### SSR support
Need to research whether Lit's experimental SSR can work with just `lit-html`. It seems to leave part UUIDs as HTML comments for this purpose.

### Debugging and extensibility
Considering littering the `Viewable` class and decorator functions with lifecycle hooks after all.

Specific advantages besides custom element classes:
* you could optionally import/enable a debug library that populates `console.debug` messages via the hooks
* you could use the hooks to extend functionality and interoperability with other libraries/APIs (or even one's own controllers)

## Not documented here
* **Styling**—constructable stylesheets are recommended though
* **Template bindings**—refer to `lit-html` documentation
