# Chapter 15: Writing Web Apps with the Raw Protocol

Most ZuzuScript programs in this guide have started at the command line:
the runtime loads a file, optionally calls `__main__`, and the script
prints or writes whatever it needs.

A raw web application has a different entrypoint. It is still an
ordinary `.zzs` file, but it defines a function called `__request__`.
The host loads the app, waits for HTTP requests, and calls that function
for each request:

```zzs
function __request__ ( env ) {
	return [
		200,
		{ "Content-Type": "text/plain; charset=UTF-8" },
		[ "Zia says hello from the den.\n" ],
	];
}
```

The return value is deliberately small:

```zzs
[ status, headers, body ]
```

This is the same basic shape used by Plack, Rack, and other small web
interfaces. The first chapter on web apps stays close to that raw
protocol. Later helper modules can make routing, cookies, forms, and
responses more comfortable, but the protocol underneath is this
three-item array.

The examples here use Zia the Raccoon, who has already wandered through
earlier chapters, as the owner of a tiny web app.


## 15.1 The Smallest Useful App

Save this as `zia-web.zzs`:

```zzs
function __request__ ( env ) {
	let path := env.get( "path" );

	if ( path == "/" ) {
		return [
			200,
			{ "Content-Type": "text/plain; charset=UTF-8" },
			[ "Zia's den is open.\n" ],
		];
	}

	if ( path == "/snacks" ) {
		return [
			200,
			{ "Content-Type": "text/plain; charset=UTF-8" },
			[
				"Snack inventory\n",
				"- berries\n",
				"- biscuits\n",
				"- suspiciously shiny wrappers\n",
			],
		];
	}

	return [
		404,
		{ "Content-Type": "text/plain; charset=UTF-8" },
		[ "Zia cannot find that trail.\n" ],
	];
}
```

The host calls `__request__` with one argument, `env`. That value is a
`Dict` containing request details. The app chooses a status code,
headers, and body, then returns them as an array.

`__main__` is not the web entrypoint. A web host parses and loads the
application, but requests go to `__request__`.


## 15.2 Running with zuzu-rust-server

The Rust web server embeds the Rust parser and runtime. To run `zia-web.zzs`
file, use the following command:

```sh
zuzu-rust-server --listen 127.0.0.1:3000 zia-web.zzs
```

Then open:

```text
http://127.0.0.1:3000/
http://127.0.0.1:3000/snacks
```

Before starting a server, you can check that the app loads and defines
`__request__`:

```sh
zuzu-rust-server --check zia-web.zzs
```

Useful development options include:

```sh
zuzu-rust-server --reload --access-log-format json zia-web.zzs
```

`--reload` watches the app file and swaps in a new worker pool after a
successful reload. `--access-log-format json` writes structured access
log lines. By default, workers are recycled after a bounded number of
requests, so long-lived memory growth in one worker does not build up
forever.


## 15.3 Running with zuzu-plackup

The Perl implementation exposes the same app contract through PSGI.
That means the same style of `__request__` app can run behind Plack
servers and middleware.

For `zia-web.zzs`, run:

```sh
bin/zuzu-plackup -Imodules zia-web.zzs -- -p 5000
```

The `--` separates ZuzuScript-side options from options passed to
`plackup`. In the example above, `-Imodules` adds the ZuzuScript module
directory, while `-p 5000` is passed through to Plack.

You can also validate the app without starting a server:

```sh
bin/zuzu-plackup -Imodules --check zia-web.zzs
```

For deployment shapes that need an ordinary `.psgi` file, use the
existing example as the model:

```sh
plackup examples/10_web_psgi_app.psgi
```

A `.psgi` wrapper can enable Plack middleware, choose a different Plack
server, or fit into an existing Perl deployment environment. The
ZuzuScript app still only needs to define `__request__`.


## 15.4 Reading the Request

The request environment is a `Dict`. The keys you will usually reach
for first are:

- `method`: HTTP method, such as `"GET"` or `"POST"`,
- `path`: decoded request path, such as `"/snacks"`,
- `query_string`: raw query string without the leading `?`,
- `headers`: request headers as a `PairList`,
- `body`: raw request body as a `BinaryString`,
- `body_text`: UTF-8 decoded request body, or `null` if it is not valid
  UTF-8.

This app echoes a small amount of request information:

```zzs
function __request__ ( env ) {
	let method_name := env.get( "method" );
	let path := env.get( "path" );
	let query := env.get( "query_string" );

	let text := "Zia checked the trail log.\n";
	text := text _ "method: " _ method_name _ "\n";
	text := text _ "path: " _ path _ "\n";
	text := text _ "query: " _ query _ "\n";

	return [
		200,
		{ "Content-Type": "text/plain; charset=UTF-8" },
		[ text ],
	];
}
```

There are also host and client metadata keys such as `scheme`, `host`,
`server_name`, `server_port`, `remote_addr`, and `raw_path`. Treat those
as server boundary details. Use the simpler keys above for most routing
and form handling.

Request bodies are read into memory by the current web hosts. That is
fine for small forms, JSON payloads, and webhook-style requests. Do not
design raw-protocol apps around unbounded uploads.


## 15.5 Returning Responses

`status` is an HTTP status code as a number:

```zzs
200
404
500
```

`headers` can be a `Dict`:

```zzs
{ "Content-Type": "text/plain; charset=UTF-8" }
```

Use a `PairList` when duplicate header names or header order matter:

```zzs
{{
	"Set-Cookie": "visited_den=1; Path=/",
	"Set-Cookie": "snack_count=3; Path=/",
	"Content-Type": "text/plain; charset=UTF-8",
}}
```

The body can be a `String`, a `BinaryString`, an array of string and
binary chunks, `null`, or a top-level `std/io` `Path` object.

An array body is useful for simple chunk composition:

```zzs
return [
	200,
	{ "Content-Type": "text/plain; charset=UTF-8" },
	[
		"Zia's report\n",
		"Door: open\n",
		"Snacks: guarded\n",
	],
];
```

Do not rely on the host to stringify arbitrary values. Convert values to
text yourself, or return a supported response body.


## 15.6 Serving a File

A top-level `Path` response body serves that file:

```zzs
from std/io import Path;

function __request__ ( env ) {
	let path := env.get( "path" );

	if ( path == "/map" ) {
		return [
			200,
			{{}},
			new Path( "docs/web-server-sketch.txt" ),
		];
	}

	return [
		404,
		{ "Content-Type": "text/plain; charset=UTF-8" },
		[ "Zia has no map for that path.\n" ],
	];
}
```

If the app does not set `Content-Type`, the host infers one from the
file extension where it can. Missing files become `404 Not Found`, and
directories become `403 Forbidden`.

A `Path` is only special as the whole response body. Do not put `Path`
objects inside a body chunk array.


## 15.7 State, Workers, and Portability

Top-level values are loaded once per app instance. That makes small
caches and counters possible:

```zzs
let visits := 0;

function __request__ ( env ) {
	visits := visits + 1;

	return [
		200,
		{ "Content-Type": "text/plain; charset=UTF-8" },
		[ "Zia has counted " _ visits _ " visits.\n" ],
	];
}
```

Treat that state as local to the current host process or worker. The
Rust server has a worker pool, and PSGI deployments may use preforking,
threads, or multiple processes. A request handled by another worker may
see a different copy of top-level state.

For portable apps, keep durable state in files, databases, or external
services. Use top-level state for things that are safe to lose or
rebuild.


## 15.8 What the Raw Protocol Is For

The raw protocol is useful because it is explicit:

- request data comes in through `env`,
- response data leaves as `[ status, headers, body ]`,
- the same app shape can run through `zuzu-rust-server` or PSGI,
- there is no hidden framework behaviour to learn first.

It is also intentionally low-level. A larger app will quickly want
helpers for routing, query parsing, forms, cookies, redirects, JSON
responses, and response construction.

That is the job of the next web chapter. It will build on this raw
protocol instead of replacing it: helper modules can make Zia's routes
tidier, but the host still calls `__request__( env )` and still expects
`[ status, headers, body ]` back.
