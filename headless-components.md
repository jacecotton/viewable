A headless component is how we implement "reactive controllers". It's just an object whose properties and methods are usable by a viewable host component (via `@controller` registration).

It's "headless" because in this case the component is not a custom element, has no lifecycle awareness, and cannot render a view. It's strictly a collection of `@state` properties, `@action` methods, and `@effect` methods. Its behavior is initialized by its own constructor, which is invoked during the host's `connectedCallback` given its prior registration with a `@controller` field.

It "renders" something simply by setting a public property that's observable by a reactive rendering entity (i.e. a viewable component's views and effects).

## Questions
* To what extent is the headless component aware of the host's context? It should neither duplicate nor implement any eventual *context API* per se (as between parent and child elements), but should it be able to read or set the host's state, or invoke any action methods? Doing so should be theoretically safe, but my first instinct is that it'd be a misuse of what's intended. Controllers should set their own state with their own action methods and run their own effects in response; but that prompts the question of what a headless component's effects would even do? In this case it wouldn't manipulate DOM, it'd basically strictly be an abstraction layer for interacting with browser APIs (timers, observers, etc.) and other external resources (global state, network interactions, etc.)
* From above: *"It "renders" something simply by setting a public property that's observable by a reactive rendering entity"* — should this replace custom directives then?
