# Routing

DLPR uses a **data-driven router** — routes are defined in a JSON file, not in code. The router reads this file at startup and matches incoming URIs to controller/method pairs. No annotations, no chained method calls, no magic.

## Route Definition

Routes live in `Data/routes.json`:

```json
{
    "/": { "controller": "PageController", "method": "home" },
    "/about": { "controller": "PageController", "method": "about" },
    "/studies": { "controller": "StudiesController", "method": "index" },
    "/studies/{id}": { "controller": "StudiesController", "method": "view" },
    "/studies/{id}/edit": { "controller": "StudiesController", "method": "edit" },
    "/studies/save": { "controller": "StudiesController", "method": "save" }
}
```

Each entry maps a URI pattern to a controller class and method name. That's the entire contract.

## How the Router Matches a Request

Given an incoming URI, the router works through three steps in order:

**1. Alias resolution** — check `Data/aliases.json` for a mapped URI and substitute it before matching. See [ROUTE_ALIASES.md](ROUTE_ALIASES.md).

**2. Exact match** — look up the URI directly in the routes array. Fast O(1) lookup.

**3. Pattern match** — if no exact match, iterate routes with `{param}` placeholders and test each against the URI via regex. Captured parameters are injected into `$request['params']`.

If nothing matches, the router returns null and the application handles a 404.

## URI Parameters

Parameters are declared with curly braces in the route key:

```json
"/studies/{id}": { "controller": "StudiesController", "method": "view" }
```

The router extracts the captured value and makes it available to the controller:

```php
public function view(array $request): array {
    $id = $request['params']['id'];
    // ...
}
```

Optional segments use bracket notation:

```json
"/user[/{id}]": { "controller": "ProfileController", "method": "view" }
```

This matches both `/user` and `/user/42`.

## Controller Resolution

The router returns a `controller` and `method` string. The application instantiates the controller class and calls the method, passing the full `$request` array:

```php
$route = $this->router->match($request['uri'], $request);
$controller = $this->instantiateController($route['controller']);
$output = $controller->{$route['method']}($request);
```

Controllers are resolved from the `App/Routes/` directory by class name. Dependencies are injected at instantiation — controllers never reach for globals or static state.

## API Routes

Routes are accessible as API endpoints by prefixing the URI with `/api/`. The router strips the prefix, resolves the route normally, and sets `$request['is_api'] = true`. The controller doesn't change — content negotiation at the response layer handles JSON vs. HTML output.

## Adding a Route

1. Add the entry to `Data/routes.json`
2. Create or extend the controller class in `App/Routes/`
3. No registration, no rebuilding — the router reads the file on the next request

## Related Docs

- [ROUTE_ALIASES.md](ROUTE_ALIASES.md) — URI shortcuts and legacy URL support
- [ENTRY_POINTS.md](ENTRY_POINTS.md) — How URI type classification happens before routing
- [LIFE_OF_REQUEST.md](LIFE_OF_REQUEST.md) — Full request trace including route matching
- [DLPR_PATTERN.md](DLPR_PATTERN.md) — Where Routes fits in the overall pattern
