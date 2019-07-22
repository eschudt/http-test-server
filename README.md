# HTTP Test Server

[![Documentation](https://docs.rs/http-test-server/badge.svg)](https://docs.rs/http-test-server/) [![Build Status](https://travis-ci.org/viniciusgerevini/http-test-server.svg?branch=master)](https://travis-ci.org/viniciusgerevini/http-test-server)

Programatically create end-points that listen for connections and return pre-defined responses.

- Allows multiple endpoints and simultaneous client connections
- Resource builder for creating endpoints
- Streaming support
- Helper functions to retrieve data such as request count, number of connected clients and
requests metadata
- Automatically allocates free port and close server after use

*NOTE*: This is not intended to work as a full featured server. For this reason many validations
and behaviours are not implemented. e.g: A request with `Accept` header with not supported
`Content-Type` won't trigger a `406 Not Acceptable`.

As this crate was devised to be used in tests, smart behaviours could be confusing and misleading.

Having said that, there are some default behaviours implemented:

- Server returns `404 Not Found` when requested resource was not configured.
- Server returns `405 Method Not Allowed` when trying to reach resource with different method from those configured.
- When a resource is created it responds to `GET` with `200 Ok` by default.

# Example:

Accept POST requests:
```rust
extern crate http_test_server;

use http_test_server::{TestServer, Resource};
use http_test_server::http::{Status, Method};

let server = TestServer::new().unwrap();
let resource = server.create_resource("/some-endpoint/new");

resource
    .status(Status::Created)
    .method(Method::POST)
    .header("Content-Type", "application/json")
    .header("Cache-Control", "no-cache")
    .body(String::from("{ \"message\": \"this is a message\" }"));

// request: POST /some-endpoint/new

// HTTP/1.1 201 Created\r\n
// Content-Type: application/json\r\n
// Cache-Control: no-cache\r\n
// \r\n
// { "message": "this is a message" }
```

Expose a persistent stream:
```rust
let server = TestServer::new().unwrap();
let resource = server.create_resource("/sub");

resource
    .header("Content-Type", "text/event-stream")
    .header("Cache-Control", "no-cache")
    .stream()
    .body(String::from(": initial data"));

// ...

resource
    .send(String::from("some data"))
    .send(String::from(" some extra data\n"))
    .send_line(String::from("some extra data with line break"))
    .close_open_connections();

// request: GET /sub

// HTTP/1.1 200 Ok\r\n
// Content-Type: text/event-stream\r\n
// Cache-Control: no-cache\r\n
// \r\n
// : initial data
// some data some extra data\n
// some extra data with line break\n
```

Redirects:
```rust
let server = TestServer::new().unwrap();
let resource_redirect = server.create_resource("/original");
let resource_target = server.create_resource("/new");

resource_redirect
    .status(Status::SeeOther)
    .header("Location", "/new" );

resource_target.body(String::from("Hi!"));

// request: GET /original

// HTTP/1.1 303 See Other\r\n
// Location: /new\r\n
// \r\n
```

Check `/tests/integration_test.rs` for more usage examples.
