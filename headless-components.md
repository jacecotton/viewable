A headless component is how we implement "reactive controllers". It's just an object whose properties and methods are usable by a viewable host component (via `@controller` registration).

It's "headless" because in this case the component is not a custom element, has no lifecycle awareness, and cannot render a view. It's strictly a collection of `@state` properties, `@action` methods, and `@effect` methods. Its behavior is initialized by its own constructor, which is invoked during the host's `connectedCallback` given its prior registration with a `@controller` field.

It "renders" something simply by setting a public property that's observable by a reactive rendering entity (i.e. a viewable component's views and effects).

The headless component is aware of the host's context, but cannot set properties on it directly—instead, like a view, it only gets a bound reference to the host's `@action` methods, which it can invoke (unlike a view).
