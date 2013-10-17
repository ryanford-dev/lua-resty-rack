# lua-resty-rack

A simple and extensible HTTP server framework for [OpenResty](http://openresty.org), providing a clean method for loading Lua HTTP applications ("resty" modules) into [Nginx](http://nginx.org).

Drawing inspiration from [Rack](http://rack.github.com/) and also [Connect](https://github.com/senchalabs/connect), **lua-resty-rack** allows you to load your application as a piece of middleware, alongside other middleware. Your application can either; ignore the current request, modify the request or response in some way and pass on to other middleware, or take responsibiliy for the request by generating a response.

## Status

This library is considered experimental and the API may change without notice. Please feel free to offer suggestions or raise issues here on Github.

## Installation

Copy the `rack.lua` file inside your nginx's `lib` folder and make sure it is included in the `lua_package_path` variable in `nginx.conf`.

## Using Middleware

To install middleware for a given `location`, you simply call `rack.use(middleware)` in the order you wish the modules to run, and then finally call `rack.run()`.

```nginx
server {
  location / {
    content_by_lua '
      local rack = require "rack"

      rack.use(require "my.middleware")
      local response = rack.run()
      rack.respond(response)
    ';
  }
}
```

### `rack.use(middleware, ...)`

The `middleware` parameter must be a callable object (usually a Lua function).

The function should accept at least two params:`req` and `next_middleware`. `req` is the request object (see below). `next_middleware`, when executed, will invoke the next middleware in the pipeline and
will return a `res` (response) object.

All middleware functions should at least return a response. It is recommended to use `res`.

```lua
rack.use(function(req, next_middleware)
  res.headers["X-Homer"] = "D'oh!"
  return next_middleware()
end)
```
Any extra arguments passed to `rack.use` will be passed to the middleware after `req` and `res`.

```lua
local replacer_mw = function(req,next_middleare,from,to)
  local res = next_middleware()
  res.body = res.body:gsub(from, to)
  return res
end

rack.use(function(req, next_middleware)
  local res = next_middleware()
  res.status = 200
  res.body   = "I like Chocolate"
  return res
end)

rack.use(body_replacer, "Chocolate", "Vanilla")
```

It's possible to chain more than one middleware by calling `rack.use` several times.
The middlewares will be executed in the same order they are included by `rack.use`. They can
modify `req` and `res` by changing their properties or adding new ones. It is required that
at least `res.status` is set by some middleware.

### `local response = rack.run()`

Runs each of the middlewares in order, until the list is finished or one of the middlewares stops the pipeline (see below).

Middlewares will be executed in the same order as they were included by `rack.use()`.

Each middleware can make modifications to `req` and `res`, and the next middleware will receive them (`res` must be returned).
The rest of parameters are optional and will be the same ones provided in `rack.use`.

The middleware pipeline can be halted by any middleware who whishes to do so, by not calling `next_middleware`.

Note that `res.status` is mandatory. Attempting to halt the pipeline without setting it will result in an error. If `res.status`
is "valid" (for example, 200), then `res.body` must be set to a non-empty string.

## `res` attributes

### `res.status`

The HTTP status code to return.
There are [constants defined](http://wiki.nginx.org/HttpLuaModule#HTTP_status_constants) for common statuses.
This value *must* be set by at least one middleware, otherwise the execution of `rack.run` will result in an error.

### `res.headers`

A table containing the request headers. Keys are matched case insensitvely, and optionally with underscores instead of hyphens. e.g.

```lua
req.headers["X-Foo"] = "bar"
res.body = req.headers.x_foo --> "bar"
```

### `res.body`

The response body. It's initially empty, and will be returned by nginx as response after the last middleware in the chain has been executed.

## `req` attributes

### `req.method`

The HTTP method, e.g. `GET`, set from `ngx.var.request_method`.

### `req.scheme`

The protocol scheme `http|https`, set from `ngx.var.scheme`.

### `req.uri`

e.g. `/my/uri`, set from `ngx.var.uri`.

### `req.host`

The hostname, e.g. `example.com`, set from `ngx.var.host`.

### `req.query`

The querystring, e.g. `var1=1&var2=2`, set from `ngx.var.query_string`.

### `req.args`

The query args, as a `table`, set from `ngx.req.get_uri_args()`.

### `req.header`

A table of headers. Just like in `res.header`, keys are normalized before being matched.

### `req.body`

The request body. It's loaded automatically (via metatables) the first time it's requested, since it's an expensive operation. Then it is
cached. It can also be set to anything else by any middleware.

## `rack.respond(response)`

Sends the response to the server. Usually `response` was returned by `rack.run`.

`rack.run` and `rack.respond` are separated so that further actions can be done in
the server after running the middlewares but before sending the response back (for
example, handling errors or storing the final response on a database). If no such
treatment is needed, the following one-liner can be used:

    rack.respond(rack.run())

## Tests

In order to execute the tests, run the following command:

    make

The tests assume that you have PERL and openresty installed.

You might need to edit the Makefile to point it to your openresty folder.


## Authors

* James Hurst (james@pintsized.co.uk)
* Raimon Grau (raimonster@gmail.com)
* Enrique García (kikito@gmail.com)

Licensed under the 2-clause BSD license. See BSD-LICENSE.md for details.
