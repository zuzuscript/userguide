# Chapter 22: Routing Requests and Building Responses

Chapter 21 showed the raw web protocol:

```zzs
function __request__ ( env ) {
	return [ 200, { "Content-Type": "text/plain" }, [ "hello\n" ] ];
}
```

That protocol is still the boundary with `zuzu-rust-server` and
`zuzu-plackup`. For larger applications, the `std/web` module adds a
friendlier layer on top of it:

```zzs
from std/web import Request, Response, Routes;
```

`Request` wraps the incoming environment, `Response` builds the outgoing
three-item response array, and `Routes` dispatches paths to actions. The
module is a Pure Zuzu Module, so it is loaded, parsed, and evaluated like
ordinary ZuzuScript code.


## 22.1 A Routed App

The smallest routed app creates a router once, then uses it from
`__request__`:

```zzs
from std/web import Request, Routes;

let routes := new Routes();

routes.get("/").to(
	action: fn req -> "The home page is open.\n",
);

routes.get("/hello/:name").to(
	action: fn req -> [ "Hello ", req.param("name"), ".\n" ],
);

function __request__ ( env ) {
	return routes.dispatch( new Request( env: env ) );
}
```

The host still calls `__request__(env)`. The difference is that the app
turns `env` into a `Request`, and `Routes.dispatch` returns the raw
response array that the host already understands.

Route actions receive one argument: the `Request`. They may return a
`Response`, a raw `[ status, headers, body ]` array, a string, a binary
string, a `Path`, a body chunk array, or `null`.

Simple return values are normalized into responses:

- a string or binary string becomes a `200` response body,
- an array that is not already `[ status, headers, body ]` becomes the
  response body,
- `null` becomes `204 No Content`,
- a `Response` is finalized directly.


## 22.2 Reading Requests

`Request` is modelled on `Plack::Request`, but it is built from the
portable Zuzu web environment.

```zzs
function inspect ( req ) {
	let text := "";
	text _= "method: " _ req.request_method() _ "\n";
	text _= "path: " _ req.path() _ "\n";
	text _= "query: " _ req.query_string() _ "\n";
	text _= "agent: " _ req.user_agent() _ "\n";
	return text;
}
```

The HTTP method accessor is named `request_method()` because `method` is
a ZuzuScript keyword. The raw value is also available as
`req.env().get("method")`.

Common request methods include:

- `env()`: the original environment `Dict`,
- `request_method()`: HTTP method, such as `"GET"` or `"POST"`,
- `path()` and `path_info()`: the route path,
- `raw_path()`: the raw path where the host provides it,
- `request_uri()`: path plus query string,
- `query_string()`: raw query string without `?`,
- `scheme()`, `secure()`, `uri()`, and `base()`,
- `headers()` and `header(name)`,
- `content_type()`, `content_length()`, `referer()`, and `user_agent()`,
- `body()`, `content()`, `raw_body()`, and `body_text()`,
- `address()`, `remote_host()`, and `user()`.

Headers are case-insensitive when read through `header(name)`.


## 22.3 Parameters and Cookies

Query strings and form bodies are parsed into `PairList` values. That
preserves ordering and duplicate keys.

```zzs
routes.post("/search").to(
	action: function ( req ) {
		let q := req.param("q");
		let all_tags := req.parameters().get_all("tag");
		return [ "search=", q, " tags=", all_tags.length(), "\n" ];
	},
);
```

`Request` provides:

- `query_parameters()`: values from the URL query string,
- `body_parameters()`: values from an
  `application/x-www-form-urlencoded` request body,
- `parameters()`: query, body, and route captures merged in that order,
- `param(name)`: the first merged value for a name,
- `param()`: all merged parameter names,
- `cookies()`: request cookies as a `Dict`.

Route captures are available through `param`, `parameters`, `captures`,
and `stash`:

```zzs
routes.get("/users/:id").to(
	action: fn req -> `user id: ${req.param("id")}\n`,
);
```

The current hosts read request bodies into memory. This is fine for
small forms, JSON payloads, and webhooks. Do not use it for unbounded
uploads.


## 22.4 Building Responses

`Response` is modelled on `Plack::Response`:

```zzs
from std/web import Response;

function created () {
	let res := new Response(
		status: 201,
		headers: { "Content-Type": "text/plain; charset=UTF-8" },
		body: [ "created\n" ],
	);
	res.header( "X-App", "example" );
	return res;
}
```

Useful methods include:

- `status(value?)` and `code(value?)`,
- `headers(value?)`,
- `header(name, value?)`,
- `body(value?)` and `content(value?)`,
- `render(template, data := {})`,
- `render_json(data)`,
- `session(value?)`,
- `content_type(value?)`,
- `content_length(value?)`,
- `content_encoding(value?)`,
- `location(value?)`,
- `redirect(url, status := 302)`,
- `set_cookie(name, value, options := {})`,
- `finalize()`.

`finalize()` returns the raw `[ status, headers, body ]` array. You
normally do not need to call it yourself when returning from a route
action, because `Routes.dispatch` normalizes `Response` objects.

Use a `PairList` when response header order or duplicates matter:

```zzs
let res := new Response( status: 200, body: [ "ok\n" ] );
res.header( "Content-Type", "text/plain; charset=UTF-8" );
res.set_cookie( "session", token, { Path: "/", HttpOnly: true } );
return res;
```

`std/web` also defines session contracts for applications which want
portable server-side sessions. Configure a handler once, read the
session from the request, and pass the finalized session to the
response:

```zzs
from std/web import Request, Response;
from std/web/session import FileSessionHandler;
from std/io import Path;

Request.set_session_handler(
	new FileSessionHandler(
		dir:     new Path( path: "/tmp/zsessions" ),
		secret:  "change-me",
		max_age: 6 * 60 * 60,
	),
);

function __request__ ( env ) {
	let req := new Request( env: env );
	let sess := req.session();
	sess{data}.set( "seen", sess{data}.get( "seen", 0 ) + 1 );
	return new Response(
		session: sess.finalize(),
		body: [ "seen ", sess{data}{seen}, "\n" ],
	);
}
```

`SessionHandler` and `Session` live in `std/web`, because `Request` and
`Response` enforce those contracts. `std/web/session` imports `std/web`
and provides `FileSessionHandler` and `DbSessionHandler`. The default
cookie is named `zzsession`; it stores only a signed opaque id. The
server-side session data is trusted marshalled storage, not encrypted or
separately signed.

`redirect` sets the status and `Location` header:

```zzs
return new Response().redirect("/login");
```

`render` fills a `std/template/z` template and sets the response body to
the rendered string. The template can be a `ZTemplate`, another object
with the same `process(data)` interface, or a `std/io` `Path`. Path
templates are compiled as `ZTemplate` objects and cached with
`std/cache/lru`:

```zzs
from std/io import Path;

return new Response()
	.content_type("text/html; charset=UTF-8")
	.render(
		new Path("templates/user.zt"),
		{ user: user },
	);
```

`render_json` encodes data as JSON, sets the response body, and sets the
content type to `application/json; charset=UTF-8`:

```zzs
return new Response().render_json({
	ok: true,
	user: user,
});
```


## 22.5 Route Patterns

Routes are checked in definition order. Matching stops at the first
route that fits the path and HTTP method.

```zzs
routes.get("/articles/:id").to(action: show_article);
routes.post("/articles").to(action: create_article);
routes.any("/health").to(action: fn req -> "ok\n");
```

Supported HTTP helpers include:

```zzs
get post put patch delete options head any
```

Standard placeholders use `:name` and match one path segment, excluding
dots:

```zzs
routes.get("/users/:id").to(action: show_user);
```

Angle brackets are another spelling for a whole-segment standard
placeholder:

```zzs
routes.get("/users/<id>").to(action: show_user);
```

Relaxed placeholders use `#name` and allow dots:

```zzs
routes.get("/assets/#filename").to(action: asset);
```

Wildcard placeholders use `*name` and can capture multiple path
segments:

```zzs
routes.get("/download/*path").to(action: download);
```

Typed placeholders use `<name:type>`. The built-in `num` type matches
non-negative whole numbers:

```zzs
routes.get("/orders/<id:num>").to(action: show_order);
```

Add custom types with `add_type`:

```zzs
routes.add_type( "slug", /^[a-z0-9-]+$/ );
routes.get("/posts/<slug:slug>").to(action: show_post);
```

When a route has the right path but the wrong method,
`Routes.dispatch` returns `405 Method Not Allowed` with an `Allow`
header. When no route matches, it returns `404 Not Found`.


## 22.6 Nested Routes

Use `under` for a shared path prefix:

```zzs
let api := routes.under("/api");

api.get("/status").to(action: fn req -> "ok\n");
api.get("/users/:id").to(action: show_api_user);
```

Nested routes combine the parent prefix and child path. Captures from
the parent remain available to the child:

```zzs
let account := routes.under("/accounts/:account_id");

account.get("/settings").to(
	action: fn req -> `settings for ${req.param("account_id")}\n`,
);
```

An `under` route may also have its own action. If that action returns a
false value, dispatch stops searching below it. This can be used for
small guards:

```zzs
let admin := routes.under("/admin").to(
	action: function ( req ) {
		return req.header("X-Admin") eq "yes";
	},
);

admin.get("/dashboard").to(action: admin_dashboard);
```


## 22.7 Controllers

For small apps, route actions can be functions:

```zzs
function welcome ( req ) {
	return "welcome\n";
}

routes.get("/welcome").to(action: welcome);
```

For larger apps, use controllers. A controller may be a class with a
static method:

```zzs
class Pages {
	static method about ( req ) {
		return "about\n";
	}
}

routes.get("/about").to(
	controller: Pages,
	action: "about",
);
```

It may be an object with an instance method:

```zzs
class Counter {
	let Number count := 0;

	method hit ( req ) {
		count++;
		return `hits: ${count}\n`;
	}
}

let counter := new Counter();
routes.get("/hits").to(
	controller: counter,
	action: "hit",
);
```

It may also be a class exported from another module:

```zzs
routes.get("/users/:id").to(
	controller: "app/controllers/users#Users",
	action: "show",
);
```

String controller targets have the form:

```text
module/path#ClassName
```

They are lazy loaded. Defining the route does not load the module. The
first matching dispatch calls `std/internals.load_module(module, class)`
and caches the returned class for later requests.

That keeps application startup cheap and avoids loading controllers for
routes that are never used.

The standard library uses the same lazy-controller shape for static
files. `std/web/static` reads its configuration from route defaults, so
the class can be shared by many routes:

```zzs
from std/io import Path;

routes.get("/img/*path").to(
	controller: "std/web/static#StaticHandler",
	action: "handle",
	root: new Path("public/img"),
);
```

`StaticHandler` serves `GET` and `HEAD`, rejects path traversal, sets
content type and cache validator headers, and serves directory index
files such as `index.html`. Directory listings are disabled by default;
enable them per route with `directory_indexes: true`.


## 22.8 Route Names and URL Rendering

Routes get a generated name from their path, and you can set one
explicitly:

```zzs
routes.get("/users/:id").name("user_show").to(action: show_user);
```

Find a named route with `find` or `lookup`:

```zzs
let user_route := routes.find("user_show");
let path := user_route.render( { id: 42 } );
```

`render` fills placeholders and percent-encodes values:

```zzs
say routes.find("user_show").render( { id: "Bob Smith" } );
// /users/Bob%20Smith
```

This is useful for links and redirects:

```zzs
return new Response().redirect(
	routes.find("user_show").render( { id: current_user_id } )
);
```


## 22.9 Running the Same App

The app still runs with the same commands as the raw protocol chapter:

```sh
zuzu-rust-server --listen 127.0.0.1:3000 app.zzs
```

or:

```sh
bin/zuzu-plackup -Imodules app.zzs -- -p 5000
```

Both hosts pass the same core environment fields used by `Request`,
including `method`, `protocol`, `scheme`, `host`, `server_name`,
`server_port`, `remote_addr`, `remote_host`, `remote_user`,
`script_name`, `path`, `raw_path`, `request_uri`, `query_string`,
`headers`, `body`, and `body_text`.

The raw protocol remains available. You can mix direct raw responses,
`Response` objects, and routed actions in the same application while
gradually moving from simple request handlers to a fuller router.
