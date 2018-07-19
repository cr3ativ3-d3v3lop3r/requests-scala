# Requests-Scala

Requests-Scala is a Scala port of the popular Python
[Requests](http://docs.python-requests.org/) HTTP client. Requests-Scala aims to
provide the same API and user-experience as the original Requests: flexible,
intuitive, and straightforward to use.

- [Making a Request](#making-a-request)
    - [Passing in Parameters](#passing-in-parameters)
    - [Response Content](#response-content)
- [Streaming Requests](#streaming-requests)
- [Multipart Uploads](#multipart-uploads)
- [Misc Configuration](#misc-configuration)
    - [Custom Headers](#custom-headers)
    - [Timeouts](#timeouts)
    - [Compression](#compression)
    - [Cookies](#cookies)
    - [Redirects](#redirects)
- [Sessions](#sessions)
## Making a Request
```scala
val r = requests.get("https://api.github.com/users/lihaoyi")

r.statusCode
// 200

r.headers("content-type")
// Buffer("application/json; charset=utf-8")

r.data.text
// {"login":"lihaoyi","id":934140,"node_id":"MDQ6VXNlcjkzNDE0MA==",...
```

Making your first HTTP request is simple: simply call `requests.get` with the
URL you want, and requests will fetch it for you.

You can also call `requests.post`, `requests.put`, etc. to make other kinds of
HTTP requests:

```scala
val r = requests.post("http://httpbin.org/post", data = Map("key" -> "value"))

val r = requests.put("http://httpbin.org/put", data = Map("key" -> "value"))

val r = requests.delete("http://httpbin.org/delete")

val r = requests.head("http://httpbin.org/head")

val r = requests.options("http://httpbin.org/get")
```

### Passing in Parameters

```scala
val r = requests.get(
    "http://httpbin.org/get", 
    params = Map("key1" -> "value1", "key2" -> "value2")
)
```
You can pass in URL parameters to GET requests via the `params` argument; simply
pass in a `Map[String, String]`. As seen earlier, when passing in POST or PUT
parameters, you instead need the `data` argument:

```scala
val r = requests.post("http://httpbin.org/post", data = Map("key" -> "value"))

val r = requests.put("http://httpbin.org/put", data = Map("key" -> "value"))
```

Apart from POSTing key-value pairs, you can also POST `String`s, `Array[Byte]`s,
`java.io.File`s, `java.nio.file.Path`s, and `requests.MultiPart` uploads:

```
requests.post("https://httpbin.org/post", data = "Hello World")
requests.post("https://httpbin.org/post", data = Array[Byte](1, 2, 3))
requests.post("https://httpbin.org/post", data = new java.io.File("thing.json"))
requests.post("https://httpbin.org/post", data = java.nio.file.Paths.get("thing.json"))
```

### Response Content

```scala
val r = requests.get("https://api.github.com/events")

r.statusCode
// 200

r.headers("content-type")
// Buffer("application/json; charset=utf-8")
```

As seen earlier, you can use `.statusCode` and `.headers` to see the relevant
metadata of your HTTP response. The response data is in the `.data` field of the
`Response` object. Most often, it's text, which you can decode using the `.text`
property as shown below:

```scala
r.data.text
// [{"id":"7990061484","type":"PushEvent","actor":{"id":6242317,"login":...
```

If you want the raw bytes of the response, use `r.data.bytes`


```scala
r.data.bytes
// Array(91, 123, 34, 105, 100, 34, 58, 34, 55, 57,  57, 48, 48, 54, 49, ...
```


## Streaming Requests

```scala
requests.get.stream("https://api.github.com/events")(
  onDownload = inputStream => {
    inputStream.transferTo(new java.io.FileOutputStream("file.json"))
  }
)
```

Requests exposes the `requests.get.stream` (and equivalent
`requests.post.stream`, `requests.put.stream`, etc.) functions for you to
perform streaming uploads/downloads without needing to load the entire
request/response into memory. This is useful if you are upload/downloading large
files or data blobs. `.stream` gives you three callbacks that get called in
order:

```scala
requests.get.stream("https://api.github.com/events")(
  onUpload = outputStream => {...},
  onHeadersReceived = streamHeaders => {...}
  onDownload = inputStream => {...}
)
```

- `onUpload` gives you a chance to write data to the server. You have access to
  a raw `java.io.OutputStream` to write to, and can easily upload data from
  memory, files, network, or any other data source.

- `onHeadersReceived is called after any upload is complete but before download
  starts: this gives you the metadata present in the header of the HTTP
  response, but without the `data` field (which you will have access to download
  later)

- `onDownload` gives you a chance to read data from the server. Again, you have
  access to the raw stream, this time a `java.io.InputStream`. You can download
  data however you like, saving it in memory, to files, sending it over the
  network, or to any other destination

Together, these three callbacks should make it easy for you to work with data
too big to fit in memory, while still benefiting from most of Requests' friendly
& intuitive API.

```

## Handling JSON

Requests does not provide any built-in JSON support, but you can easily use a 
third-party JSON library to work with it. This example shows how to use 
[uJson](http://www.lihaoyi.com/upickle/) talk to a HTTP endpoint that requires a 
JSON-formatted body, either using `ujson.write`:

```scala
requests.get(
  "https://api.github.com/some/endpoint",
  data = ujson.write(Map("user-agent" -> "my-app/0.0.1"))
)
```
```scala
requests.get(
  "https://api.github.com/some/endpoint",
  data = ujson.write(ujson.Js.Obj("user-agent" -> "my-app/0.0.1"))
)
```

It is equally easy ot use uJson to deal with JSON returned in the response from
the server:

```scala
val r = requests.get("https://api.github.com/events")

val json = ujson.read(r.data.text)

json.arr.length
// 30

json.arr(0).obj.keys
// Set("id", "type", "actor", "repo", "payload", "public", "created_at")
```

While Requests-Scala doesn't come bundled with JSON functionality, it is trivial
to use it together with any other 3rd party JSON library (I like
[uJson](http://www.lihaoyi.com/upickle/)) So just pick whatever library you
want.

## Multipart Uploads

```scala
val r = requests.post(
  "http://httpbin.org/post",
  data = requests.MultiPart(
    requests.MultiItem("name", new java.io.File("build.sc"), "file.txt"),
    // you can upload strings, and file name is optional
    requests.MultiItem("name2", "Hello"), 
    // bytes arrays are ok too
    requests.MultiItem("name3", Array[Byte](1, 2, 3, 4)) 
  )
)
```

Multipart uploads are done by passing `requests.MultiPart`/`requests.MultiItem`
to the `data` parameter. Each `MultiItem` needs a name and a data-source, which
can be a `String`, `Array[Byte]`, `java.io.File`, or `java.nio.file.Path`. Each
`MultiItem` can optionally take a file name that will get sent to the server

## Misc Configuration

Earlier you already saw how to use the `params` and `data` arguments. Apart from
those, the `requests.get` method takes in a lot of arguments you can use to
configure it, e.g. passing in custom headers:

### Custom Headers

```scala
requests.get(
  "https://api.github.com/some/endpoint",
  headers = Map("user-agent" -> "my-app/0.0.1")
)
```


### Timeouts

`readTimeout`s and `connectTimeout`s:

```scala
requests.get("https://httpbin.org/delay/1", readTimeout = 10)
// TimeoutException

requests.get("https://httpbin.org/delay/1", readTimeout = 1500)
// ok 

requests.get("https://httpbin.org/delay/3", readTimeout = 1500)
// TimeoutException
```

```scala
requests.get("https://httpbin.org/delay/1", connectTimeout = 10)
// TimeoutException

requests.get("https://httpbin.org/delay/1", connectTimeout = 1500)
// ok

requests.get("https://httpbin.org/delay/3", connectTimeout = 1500)
// ok
```

### Compression

Configuration for compressing the request `data` upload with Gzip or Deflate via
the `compress` parameter:

```scala
requests.post(
  "https://httpbin.org/post",
  compress = requests.Compress.None,
  data = "Hello World"
)

requests.post(
  "https://httpbin.org/post",
  compress = requests.Compress.Gzip,
  data = "I am cow"
)

requests.post(
  "https://httpbin.org/post",
  compress = requests.Compress.Deflate,
  data = "Hear me moo"
)
```

Or to disabling the de-compression of the response `data` being downloaded via
the `autoCompress` parameter, in case you want the un-compressed data blob for
whatever reason:

```scala
requests.get("https://httpbin.org/gzip").data.bytes.length
// 250

requests.get("https://httpbin.org/gzip", autoDecompress=false).data.bytes.length
// 201


requests.get("https://httpbin.org/deflate").data.bytes.length
// 251

requests.get("https://httpbin.org/deflate", autoDecompress=false).data.bytes.length
// 188
```

### Cookies

You can take the cookies that result from one HTTP request and pass them into a
subsequent HTTP request:

```scala
val r = requests.get("https://httpbin.org/cookies/set?freeform=test")

r.cookies
// Map("freeform" -> freeform=test)
```
```scala

val r2 = requests.get("https://httpbin.org/cookies", cookies = r.cookies)

r2.data.text
// {"cookies":{"freeform":"test"}}
```

This is a common pattern, e.g. to maintain an authentication/login session
across multiple requests. However, it may be easier to instead use Sessions...


### Redirects

Requests handles redirects automatically for you, up to a point:

```scala
val r = requests.get("http://www.github.com")

r.url
// https://github.com/

r.history
// Some(Response("https://www.github.com", 301, "Moved Permanently", ...

r.history.get.history
// Some(Response("http://www.github.com", 301, "Moved Permanently", ...

r.history.get.history.get.history
// None
```

As you can see, the request to `http://www.github.com` was first redirected to
`https://www.github.com`, and then to `https://github.com/`. Requests by default
only follows up to 5 redirects in a row, though this is configurable via the
`maxRedirects` parameter:

```scala
val r0 = requests.get("http://www.github.com", maxRedirects = 0)
// Response("http://www.github.com", 301, "Moved Permanently", ...

r0.history
// None

val r1 = requests.get("http://www.github.com", maxRedirects = 1)
// Response("http://www.github.com", 301, "Moved Permanently", ...

r1.history
// Some(Response("http://www.github.com", 301, "Moved Permanently", ...

r1.history.get.history
// None
```

As you can see, you can use `maxRedirects = 0` to disable redirect handling
completely, or use another number to control how many redirects Requests follows
before giving up.

All of the intermediate responses in a redirect chain are available in a
Response's `.history` field; each `.history` points 1 response earlier, forming
a linked list of `Response` objects until the earliest response has a value of
`None`. You can crawl up this linked list if you want to inspect the headers or
other metadata of the intermediate redirects that brought you to your final value.

## Sessions

A `requests.Session` automatically handles sending/receiving/persisting cookies
for you across multiple requests:

```scala
val s = requests.Session()

val r = s.get("https://httpbin.org/cookies/set?freeform=test")

val r2 = s.get("https://httpbin.org/cookies", cookies = r.cookies)

r2.data.text
// {"cookies":{"freeform":"test"}}
```

If you want to deal with a website that uses cookies, it's usually easier to use
a `requests.Session` rather than passing around `cookie` variables manually.

Apart from persisting cookies, sessions are also useful for consolidating common
configuration that you want to use across multiple requests, e.g. custom
headers, cookies or other things:

```scala
val s = requests.Session(
  headers = Map("x-special-header" -> "omg"), 
  cookieValues = Map("cookie" -> "vanilla")
)

val r1 = requests.get("https://httpbin.org/cookies")

r1.data.text
// {"cookies":{"cookie":"vanilla"}}

val r2 = requests.get("https://httpbin.org/cookies")

r1.data.text
// {"cookies":{"cookie":"vanilla"}}

val r3 = s.get("https://httpbin.org/headers")

r3.data.text
// {"headers":{"X-Special-Header":"omg", ...}}

val r4 = s.get("https://httpbin.org/headers")

r4.data.text
// {"headers":{"X-Special-Header":"omg", ...}}
"""
```

