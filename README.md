FastRoute - Fast request router for PHP
=======================================

This library provides a fast implementation of a regular expression based router. I'll write a
blog post about how the implementation works and why it is fast another time.

Usage
-----

Here's a basic usage example:

```php
<?php

require '/path/to/FastRoute/src/bootstrap.php'

$dispatcher = FastRoute\simpleDispatcher(function(FastRoute\RouteCollector $r) {
    $r->addRoute('GET', '/user/{name}/{id:[0-9]+}', 'handler0');
    $r->addRoute('GET', '/user/{id:[0-9]+}', 'handler1');
    $r->addRoute('GET', '/user/{name}', 'handler2');
});

$routeInfo = $dispatcher->dispatch($methodMethod, $uri);
switch ($routeInfo[0]) {
    case FastRoute\Dispatcher::NOT_FOUND:
        // ... 404 Not Found
        break;
    case FastRoute\Dispatcher::METHOD_NOT_ALLOWED:
        $allowedMethods = $routeInfo[1];
        // ... 405 Method Not Allowed
        break;
    case FastRoute\Dispatcher::FOUND:
        $handler = $routeInfo[1];
        $vars = $routeInfo[2];
        // ... call $handler with $vars
        break;
}
```

### Defining routes

The routes are defined by calling the `FastRoute\simpleDispatcher` function, which accepts
a callable taking a `FastRoute\RouteCollector` instance. The routes are added by calling
`addRoute()` on the collector instance.

This method accepts the HTTP method the route must match, the route pattern and an associated
handler. The handler does not necessarily have to be a callback (it could also be a controller
class name or any other kind of data you wish to associate with the route).

By default a route pattern syntax is used where `{foo}` specified a placeholder with name `foo`
and matching the string `[^/]+`. To adjust the pattern the placeholder matches, you can specify
a custom pattern by writing `{bar:[0-9]+}`. However, it is also possible to adjust the pattern
syntax by passing using a different route parser.

The reason `simpleDispatcher` accepts a callback for defining the routes is to allow seamless
caching. By using `cachedDispatcher` instead of `simpleDispatcher` you can cache the generated
routing data and construct the dispatcher from the cached information:

```php
<?php

$dispatcher = FastRoute\cachedDispatcher(function(FastRoute\RouteCollector $r) {
    $r->addRoute('GET', '/user/{name}/{id:[0-9]+}', 'handler0');
    $r->addRoute('GET', '/user/{id:[0-9]+}', 'handler1');
    $r->addRoute('GET', '/user/{name}', 'handler2');
}, [
    'cacheFile' => __DIR__ . '/route.cache', /* required */
    'cacheDisabled' => IS_DEBUG_ENABLED,     /* optional, enabled by default */
]);
```

The second parameter to the function is an options array, which can be used to specify the cache
file location, among other things.

### Dispatching a URI

A URI is dispatched by calling the `dispatch()` method of the created dispatcher. This method
accepts the HTTP method and a URI. Getting those two bits of information (and normalizing them
appropriately) is your job - this library is not bound to the PHP web SAPIs.

The `dispatch()` method returns an array those first element contains a status code. It is one
of `Dispatcher::NOT_FOUND`, `Dispatcher::METHOD_NOT_ALLOWED` and `Dispatcher::FOUND`. For the
method not allowed status the second array element contains a list of HTTP methods allowed for
this method. For example:

    [FastRoute\Dispatcher::METHOD_NOT_ALLOWED, ['GET', 'POST']]

For the found status the second array element is the handler that was associated with the route
and the third array element is a dictionary of placeholder names to their values. For example:

    /* Routing against GET /user/nikic/42 */
    
    [FastRoute\Dispatcher::FOUND, 'handler0', ['name' => 'nikic', 'id' => '42']]

### Overriding the route parser and dispatcher

The routing process makes use of three components: A route parser, a data generator and a
dispatcher. The three components adhere to the following interfaces:

```php
<?php

namespace FastRoute;

interface RouteParser {
    public function parse($route);
}

interface DataGenerator {
    public function addRoute($httpMethod, $routeData, $handler);
    public function getData();
}

interface Dispatcher {
    const NOT_FOUND = 0, FOUND = 1, METHOD_NOT_ALLOWED = 2;

    public function dispatch($httpMethod, $uri);
}
```

The route parser takes a route pattern string and converts it into an array of it's parts. The
array has a certain structure, best understood using an example:

    /* The route /user/{name}/{id:[0-9]+} converts to the following array: */
    [
        '/user/',
        ['name', '[^/]+'],
        '/',
        ['id', [0-9]+'],
    ]

This array can then be passed to the `addRoute()` method of a data generator. After all routes have
been added the `getData()` of the generator is invoked, which returns all the routing data required
by the dispatcher. The format of this data is not further specified - it is tightly coupled to
the corresponding dispatcher.

The dispatcher accepts the routing data via a constructor and provides a `dispatch()` method, which
you're already familiar with.

The route parser can be overwritten individually (to make use of some different pattern syntax),
however the data generator and dispatcher should always be changed as a pair, as the output from
the former is tightly coupled to the input of the latter. The reason the generator and the
dispatcher are separate is that only the latter is needed when using caching (as the output of
the former is what is being cached.)

When using the `simpleDispatcher` / `cachedDispatcher` functions from above the override happens
through the options array:

```php
<?php

$dispatcher = FastRoute\simpleDispatcher(function(FastRoute\RouteCollector $r) {
    /* ... */
}, [
    'routeParser' => 'FastRoute\\RouteParser\\Std',
    'dataGenerator' => 'FastRoute\\DataGenerator\\GroupCountBased',
    'dispatcher' => 'FastRoute\\Dispatcher\\GroupCountBased',
]);
```

The above options array corresponds to the defaults. By replacing `GroupCountBased` by
`GroupPosBased` you could switch to a different dispatching strategy.

