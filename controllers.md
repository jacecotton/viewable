Controllers are objects that hook into a component's reactive lifecycle, for the purpose of abstracting and sharing stateful logic and behavior across components.

* Controllers mutate host state in response to the host component's lifecycle as well as internal criteria such as event listeners, network responses, or remote store updates.
* Controllers can have their own internal state for bookkeeping through private fields, but these are not allowed to be "observable", as that implies a separate and parallel reactive lifecycle to the host. Controllers do not have their own views and do not define their own actions.
* In other words, the host owns state and lifecycle, while controllers help compose logic and behavior. Controllers are privileged internal collaborators, not autonomous components.
* If you find yourself needing the host to react to controller state, then that state should be moved up to the host(s).

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

## Implementation reminders
* Decorators should validate that the current object is a `ViewableComponent` and not a controller.
* `@effect` needs to work in `Controller`s as well. `ViewableComponent` should simply merge the controller object's `EFFECTS_COLL` with its own as part of the registration process. The controller instance's `EFFECTS_COLL` can then be trashed.
