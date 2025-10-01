Controllers are objects that hook into a component's reactive lifecycle, for the purpose of abstracting and sharing stateful logic and behavior across components.

* Controllers mutate host state in response to the host component's lifecycle as well as internal criteria such as event listeners, network responses, or remote store updates.
* Controllers can have their own internal state for bookkeeping through private fields, but these are not allowed to be "observable", as that implies a separate and parallel reactive lifecycle to the host. Controllers do not have their own views or effects, and do not define their own actions.
* In other words, the host owns state and lifecycle, while controllers help compose logic and behavior. Controllers are privileged internal collaborators, not autonomous components.
* If you find yourself needing the host to react to controller state, then that state should be moved up to the host(s).
* Likewise, controllers cannot define their own effects. Controllers *provide capabilities* (invocable methods, attached listeners and observers), they do not *implement policies* (effects, i.e. "how the host should respond to state changes").
* This establishes clear ownership, preventing hidden dependencies, making tracing data flows easier, preventing infinite loops, and improving testability.
