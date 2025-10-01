Controllers are objects that hook into a component's reactive lifecycle, for the purpose of abstracting and sharing stateful logic and behavior across components.

* Controllers mutate host state in response to the host component's lifecycle as well as internal criteria such as event listeners, network responses, or remote store updates.
* Controllers can have their own internal state for bookkeeping through private fields, but these are not allowed to be "observable", as that implies a separate and parallel reactive lifecycle to the host. Controllers do not have their own views or effects, and do not define their own actions.
* In other words, the host owns state and lifecycle, while controllers help compose logic and behavior. Controllers are privileged internal collaborators, not autonomous components.
  * If you find yourself needing the host to react to controller state, then that state should be moved up to the host(s).
* Likewise, controllers cannot define their own effects. Controllers *provide capabilities* (invocable methods, attached listeners and observers), they do not *implement policies* (effects, i.e. "how the host should respond to state changes").
  * If you find yourself needing the controller to hook into a host's effects, expose a method and invoke it in the host's effect method.
* This establishes clear ownership, preventing hidden dependencies, making tracing data flows easier, preventing infinite loops, and improving testability.

## Basic shape
```js
class SomeComponent extends ViewableComponent {
  // ViewableComponent instantiates SomeController, automatically setting
  // `this` as `foo.host`.
  @controller(SomeController) foo;
}

class SomeController extends Controller {
  @host connectedCallback() {
    console.log("Host is connected!");
  }
}
```

This introduces two new decorators:
* `@controller` — instantiates the object passed to it, makes instance properties and methods accessible through the decorated property (`foo`).
* `@host` — upstreams method to `host`'s equivalent.

## Questions
Should controllers be able to define their own effects? There are benefits and dangers.

Benefits:
* It makes sense. The whole point is a controller should be allowed to participate in the host component's reactive lifecycle.
* If you're constantly having to call controller methods in host effects, it starts to feel less like true behavioral abstraction and logic sharing, and more like pointless (and error-prone) indirection.

Dangers:
* Infinite loops — effect for `a` could mutate `b`, and effect for `b` could mutate `a`. This wouldn't be unique to controllers, but controllers with effects could compound the problem and make debugging more difficult.
* Blurred ownership — whether to put behavior in a host or controller effect is very subjective and can become difficult to keep track of. There's no real way to enforce that controller effects truly stick to a focused scope.

## Implementation reminders
* Decorators should validate that the current object is a `ViewableComponent` and not a controller.
