# Chapter 20: Web Client Requests and URLs

Chapter 19 worked with tabular data in databases and CSV files.

Those tables might come from an API, and reports might need to be uploaded
somewhere else. This chapter moves to the web from the client side:

- building URLs,
- escaping query values,
- using URL templates,
- sending `GET`, `POST`, and `HEAD` requests,
- reading response headers and bodies,
- saving remote files to the local filesystem,
- choosing synchronous or asynchronous HTTP,
- and setting request timeouts and retries.

The two main modules are:

- `std/net/url` for URL parsing, escaping, and templates,
- `std/net/http` for HTTP requests.

HTTP examples require network support. File download examples also require
filesystem support. In browser-hosted JavaScript, asynchronous HTTP is the
portable path; synchronous HTTP is not supported there.


## 20.1 URL escaping

Import the URL helpers from `std/net/url`:

```zzs
from std/net/url import escape, unescape;
```

Use `escape` for values that will be placed inside a URL:

```zzs
let query := escape("flat white near station");

say query;       // flat%20white%20near%20station
```

Use `unescape` to decode percent-escaped text:

```zzs
let text := unescape("flat%20white%20near%20station");

say text;        // flat white near station
```

Do not build query strings by hand when user input is involved:

```zzs
let city := "São Paulo";
let q := "coffee & pastry";

let url := "https://example.com/search"
	_ "?city=" _ escape(city)
	_ "&q=" _ escape(q);
```

Escaping turns reserved characters and non-ASCII text into a URL-safe
form.


## 20.2 Parsing URLs

Use `parse(url)` when you need to inspect a URL:

```zzs
from std/net/url import parse;

let parsed := parse(
	"https://user@example.com:8443/cafes?q=latte#top",
);

say parsed{scheme};              // https
say parsed{host};                // example.com
say parsed{port};                // 8443
say parsed{path};                // /cafes
say parsed{query};               // q=latte
say parsed{fragment};            // top
say parsed{query_params}{q};     // latte
```

The result is a dictionary with keys such as:

- `url`,
- `scheme`,
- `authority`,
- `userinfo`,
- `host`,
- `port`,
- `path`,
- `query`,
- `fragment`,
- and `query_params`.

`query_params` is useful when a script needs to read or modify a URL
instead of treating it as opaque text.


## 20.3 URL templates

For repeated API calls, use `fill_template`.

```zzs
from std/net/url import fill_template;

let url := fill_template(
	"https://api.example.com/{version}/cafes/{id}{?fields}",
	{
		version: "v1",
		id: 42,
		fields: "name,rating,open_now",
	},
);

say url;
```

That produces:

```text
https://api.example.com/v1/cafes/42?fields=name%2Crating%2Copen_now
```

Templates are good when the shape of the URL is stable but the values
change.

Query templates can include more than one value:

```zzs
let search_url := fill_template(
	"https://api.example.com/{version}/search{?q,city,limit}",
	{
		version: "v1",
		q: "quiet coffee",
		city: "München",
		limit: 10,
	},
);
```

`fill_template` handles the escaping for values placed into the URL.


## 20.4 A first GET request

Import `UserAgent` from `std/net/http`:

```zzs
from std/net/http import UserAgent;
```

A `UserAgent` owns request settings such as default headers, timeout, and
cookie jar.

```zzs
from std/net/http import UserAgent;

let ua := new UserAgent(
	agent: "coffee-client/1.0",
	timeout: 10,
	default_headers: {
		accept: "application/json",
	},
);

let response := ua.get("https://api.example.com/v1/cafes");

if ( response.success() ) {
	say response.content();
}
else {
	warn `HTTP ${response.status()}: ${response.reason()}`;
}
```

`response.status()` is the numeric HTTP status. `response.reason()` is the
reason text. `response.success()` is true for successful HTTP status
codes.

When a request must succeed, use `expect_success()`:

```zzs
let response := ua.get("https://api.example.com/v1/cafes");
let body := response.expect_success().content();
```

If the status is not successful, `expect_success()` throws an exception.


## 20.5 Decoding JSON responses

Many web APIs return JSON.

Use `response.json()` to decode the response body:

```zzs
let response := ua
	.get("https://api.example.com/v1/cafes")
	.expect_success();

let data := response.json();

for ( let cafe in data{items} ) {
	say cafe{name};
}
```

You can also convert the response metadata to a dictionary:

```zzs
let snapshot := response.to_Dict();

say snapshot{status};
say snapshot{url};
```


## 20.6 Query parameters with request builders

For anything beyond a very small request, use `build_request`.

```zzs
let request := ua
	.build_request("GET", "https://api.example.com/v1/search")
	.query(
		{
			q: "quiet coffee",
			city: "München",
			limit: 10,
		},
	)
	.header("accept", "application/json");

let response := ua.send(request);
let data := response.expect_success().json();
```

`query(...)` appends URL query parameters and escapes the values.

The builder methods return the request, so calls can be chained:

```zzs
let response := ua.send(
	ua
		.build_request("GET", "https://api.example.com/v1/search")
		.query( { q: "espresso", limit: 5 } )
		.auth_bearer("TOKEN")
		.timeout(5)
		.retries(1),
);
```

Use `auth_bearer(token)` for bearer-token APIs. It sets the
`Authorization` request header.


## 20.7 HEAD requests

Use `HEAD` when you need metadata without downloading the body.

```zzs
let response := ua.head("https://example.com/menu.pdf");

if ( response.success() ) {
	say response.header("content-type");
	say response.header("content-length");
}
```

Header lookup is case-insensitive:

```zzs
say response.header("Content-Type");
say response.header("content-type");
```

Both ask for the same header.

`HEAD` is useful before a download when you want to check size, content
type, cache validators, or whether the remote resource exists.


## 20.8 POST requests

For simple form-style posts, pass the body and headers:

```zzs
let response := ua.post(
	"https://api.example.com/v1/feedback",
	"name=Zia&rating=5",
	{
		"content-type": "application/x-www-form-urlencoded",
	},
);

say response.status();
```

For JSON APIs, use the request builder's `json` method:

```zzs
let request := ua
	.build_request("POST", "https://api.example.com/v1/cafes")
	.json(
		{
			name: "Moonbean",
			city: "Cardiff",
			rating: 5,
		},
	)
	.auth_bearer("TOKEN");

let response := ua.send(request);

if ( response.success() ) {
	say response.json(){id};
}
```

`json(value)` encodes the value as JSON and sets the request content type.

The generic form is also available:

```zzs
let response := ua.request(
	"POST",
	"https://api.example.com/v1/feedback",
	"name=Zia&rating=5",
	{
		"content-type": "application/x-www-form-urlencoded",
	},
);
```


## 20.9 Downloading files

To fetch a remote file and decide what to do with the bytes yourself, read
the response content:

```zzs
from std/io import Path;
from std/net/http import UserAgent;

let ua := new UserAgent(timeout: 30);
let response := ua.get("https://example.com/menu.pdf");
let bytes := response.expect_success().content();

( new Path("menu.pdf") ).spew(bytes);
```

Use binary file methods such as `spew(bytes)` for downloaded bytes. Use
UTF-8 methods such as `spew_utf8(text)` only after you have decoded or
converted the content to text.

For direct downloads, configure the request with `download_to(path)`:

```zzs
let out := new Path("menu.pdf");

let request := ua
	.build_request("GET", "https://example.com/menu.pdf")
	.download_to(out)
	.timeout(30);

ua.send(request).expect_success();
```

That streams the response into the destination file as the request runs.


## 20.10 Uploading files

Use `upload_from(path)` when the request body should come from a local
file:

```zzs
let report := new Path("coffee-report.json");

let request := ua
	.build_request("PUT", "https://api.example.com/v1/reports/today")
	.upload_from(report)
	.header("content-type", "application/json")
	.auth_bearer("TOKEN");

let response := ua.send(request);

say response.status();
```

For multipart form fields, use `multipart`:

```zzs
let response := ua.send(
	ua
		.build_request("POST", "https://api.example.com/v1/forms")
		.multipart(
			{
				title: "Coffee report",
				notes: "Lots of tables, one quiet corner.",
			},
		),
);
```

The exact server-side expectations for uploads and forms vary by API.
Match the field names and content types documented by the service you are
calling.


## 20.11 Synchronous and asynchronous HTTP

Synchronous HTTP is simple:

```zzs
let response := ua.get("https://api.example.com/v1/cafes");
let cafes := response.expect_success().json();
```

It is appropriate for short scripts where waiting for one request at a
time is acceptable.

Use asynchronous HTTP when:

- several requests can run at once,
- you are writing a server or UI that must stay responsive,
- or the browser runtime is a target.

Async methods return tasks. Await them:

```zzs
async function load_cafes () {
	let ua := new UserAgent(timeout: 10);

	let response := await {
		ua.get_async("https://api.example.com/v1/cafes");
	};

	return response.expect_success().json();
}

let cafes := await {
	load_cafes();
};
```

The async method names mirror the sync names:

- `get_async`,
- `head_async`,
- `post_async`,
- `request_async`,
- `send_async`,
- and `request.send_async(user_agent)`.


## 20.12 Fetching several URLs at once

Use `std/task` `all` to await several requests:

```zzs
from std/net/http import UserAgent;
from std/task import all;

async function load_two_pages () {
	let ua := new UserAgent(timeout: 10);

	let responses := await {
		all( [
			ua.get_async("https://api.example.com/v1/cafes"),
			ua.get_async("https://api.example.com/v1/roasters"),
		] );
	};

	return [
		responses[0].expect_success().json(),
		responses[1].expect_success().json(),
	];
}
```

Each response is still a normal `Response` object. Check status, headers,
and content the same way you would for synchronous requests.


## 20.13 Timeouts and retries

Set a default timeout (in seconds) on the user agent:

```zzs
let ua := new UserAgent(timeout: 10);
```

Override it for one request:

```zzs
let request := ua
	.build_request("GET", "https://api.example.com/v1/cafes")
	.timeout(3);
```

Use retries for idempotent requests such as `GET` and `HEAD`:

```zzs
let response := ua.send(
	ua
		.build_request("GET", "https://api.example.com/v1/cafes")
		.timeout(3)
		.retries(2),
);
```

Be careful retrying `POST`, `PUT`, or `PATCH`. If the server received the
first request but the client timed out while waiting, a retry may repeat
the change.


## 20.14 Cookies and default headers

Use a `CookieJar` when you want cookies to persist across requests made by
the same user agent:

```zzs
from std/net/http import CookieJar, UserAgent;

let jar := new CookieJar();
let ua := new UserAgent(
	cookie_jar: jar,
	default_headers: {
		accept: "application/json",
	},
);
```

The user agent stores cookies from responses and sends matching cookies on
later requests.

You can also add a cookie manually:

```zzs
jar.add(
	"https://api.example.com/",
	"session=abc123; Path=/; Secure; HttpOnly",
);

say jar.cookie_header("https://api.example.com/v1/cafes");
```

Use default headers for values every request should carry, and request
headers for values that apply to only one request.


## 20.15 A small client wrapper

For real code, wrap the user agent and base URL in functions.

```zzs
from std/net/http import UserAgent;
from std/net/url import fill_template;

const BASE := "https://api.example.com/{version}";

let ua := new UserAgent(
	agent: "coffee-client/1.0",
	timeout: 10,
	default_headers: {
		accept: "application/json",
	},
);

function api_url ( path, params := {} ) {
	return fill_template(
		BASE _ "/{path}{?q,city,limit}",
		{
			version: "v1",
			path: path,
			q: params.get("q", null),
			city: params.get("city", null),
			limit: params.get("limit", null),
		},
	);
}
```

The template omits query parameters whose values are `null`, so callers can
provide only the fields they need.

```zzs
function search_cafes ( query, city ) {
	let response := ua.get(
		api_url(
			"search",
			{
				q: query,
				city: city,
				limit: 10,
			},
		),
	);

	return response.expect_success().json();
}
```

The point is not to hide HTTP. The point is to keep URL construction,
authentication, timeouts, and response handling in one predictable place.


## 20.16 Client checklist

Before writing a web client, decide:

- whether the code can block or should be asynchronous,
- which requests are safe to retry,
- which URLs should be templates,
- which values must be escaped,
- how to handle unsuccessful HTTP statuses,
- whether response bodies are text, JSON, or bytes,
- where downloaded files should be saved,
- and whether cookies or authentication headers need to persist.

Use `std/net/url` to make URLs boring and correct. Use `std/net/http` to
make requests explicit: method, URL, headers, body, timeout, and response
handling.

So far the script has been the client. Chapter 21 turns the direction
around: ZuzuScript receives HTTP requests and returns responses.
