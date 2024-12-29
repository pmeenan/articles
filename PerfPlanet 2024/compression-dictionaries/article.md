# Getting Real (small) With Compression Dictionaries

[Compression dictionary transport](https://datatracker.ietf.org/doc/draft-ietf-httpbis-compression-dictionary/) is a relatively new feature in HTTP that allows for using custom compression dictionaries to improve the compression of HTTP responses. The results can be pretty dramatic, with responses anywhere from 60-90+% smaller than using the best non-dictionary compression currently available. The feature has been under experimentation in Chrome over the last two years and shipped to stable earlier this year in Chrome 130. This article will discuss how to implement and ship a production-quality implementation that uses dictionary compression on the edge without having to modify an existing application (based on what we learned with several implementations that have shipped or are getting ready to ship).

For more background on compression dictionary transport I recommend reading the web.dev [article](https://developer.chrome.com/blog/shared-dictionary-compression) on the topic and watching my [video](https://youtu.be/Gt0H2DxdAPY?si=Nzw-DehzUhtSPhAa) from the performance.now() 2023 conference.

## Using compression dictionaries

The core feature of compression dictionaries is that any HTTP response can be configured to be used as a dictionary for future requests. This is done by adding a `Use-As-Dictionary:` header to the response with [instructions](https://www.ietf.org/archive/id/draft-ietf-httpbis-compression-dictionary-19.html#name-use-as-dictionary) to the client on what requests to use the dictionary for. The instructions include a URLPattern (`match`) for URLs where the dictionary should match but can also include a destination filter (`match-dest`) to restrict it to a specific type of request (e.g. documents or scripts). The instructions also allow for an opaque `id` to be set for the dictionary that will be echoed back in requests.

Practically, there are two common use-cases where they can be used:

1. Delta-updates for static resources:

In this case, a given response will be used as the dictionary for an updated version of the same resource. For example, `/scripts/v1/common.js` could be used at some point in the future as a dictionary for `/scripts/v2/common.js`. This effectively compresses away everything except for what changed in the update, leading to a MUCH smaller response. Each static resource would be a separate dictionary for newer versions of itself. This is also very low-overhead since the original file is already being downloaded to be used as part of loading the application so it just requires an additional header to let the browser know that it should also be used as a dictionary:

```
Use-As-Dictionary: id="/scripts/v1/common.js", match="/scripts/*/common.js", match-dest=("script")
```

2. Stand-alone dictionaries:

In the stand-alone case, you [generate](https://use-as-dictionary.com/generate/) a custom dictionary for the content you are intending to use it with and request that the browser load the dictionary. The most common use case for this is to generate a dictionary for the HTML for a site to compress-out all of the common content (HTML includes, headers, footers, contents, structure, etc). It can also be useful for API calls, particularly when JSON is used, to compress-out all of the common response fields.

This can be quite effective, usually leading to 60-90% smaller HTML responses and it is in the critical path of each page load so it can be quite impactful but it does require downloading the dictionary which will use some additional bytes, usually during an idle period.

A `fetch`, `preload` or `prefetch` could be used to trigger the dictionary fetch since it is the response that configures the use as a dictionary but the spec also provides for a new `link` type of `compression-dictionary` that can be used (and lets browsers prioritize the fetch appropriately).

in HTML:

```HTML
<link rel="compression-dictionary" href="/dictionary/v1.dict">
```

or as an HTTP response header:

```
Link: </dictionary/v1.dict>; rel="compression-dictionary"
```

Then, in the response for the dictionary (`/dictionary/v1.dict` in this case), the dictionary instructions would be provided:

```
Use-As-Dictionary: id="/dictionary/v1.dict", match="/*", match-dest=("document" "frame")
```

In this case, the dictionary will be applied to all document (and frame) requests from the same origin.

## Implementation

I have an example implementation [here](https://github.com/pmeenan/dictionary-worker/blob/main/src/index.js) of a Cloudflare worker that, with a bit of configuration, implements both the delta-update and document compression use-cases. You can see it operating live on a wordpress blog [here](https://roadtrip.meenan.us/).

I used the `id` field to store the URL for the original dictionary so that the compression side of things can be completely stateless and fetch the dictionaries as-needed (this does require that all resources used as dictionaries will be immutable and use permanent URLs). This allows us to cleanly separate the logic into two parts:

1. [Adding the relevant headers](#adding-the-headers) to responses as needed.
2. [Dictionary-compressing](#compressing-the-responses) the responses.

### Adding the headers

There are two sets of headers that need to be added. The `Link` header to trigger the stand-alone dictionary fetches and the `Use-As-Dictionary` header to configure a given response as a dictionary.

For the `Link` header, it is relatively simple. If a given request comes in with a `Sec-Fetch-Dest: document` request header, then I add `Link` headers for all of the configured stand-alone dictionaries.

```Javascript
const dynamic_dictionaries = [
  {
    "url": "/dictionary/v1.dict",
    "match": "/*",
    "match-dest": ["document", "frame"],
  }
]
```

The worker code can just loop through the list of `dynamic_dictionaries` and use the `url` specified for each link tag (only one in my case but there can be multiple).

For the `Use-As-Dictionary` header, the worker needs to identify relevant responses and add the dictionary header with appropriate instructions to each one. With application-specific knowledge this could be as simple as looking for everything in a given path but for the example worker it takes a configured list of URLPatterns of resources:

```Javascript
let static_dictionaries = {
  "script": [
    "/wp-includes/js/jquery/jquery.min.js?ver=*",
    "/wp-includes/js/jquery/jquery-migrate.min.js?ver=*",
    "/*/fontawesome/js/all.min.js?ver=*",
  ],
  "style": [
    "/wp-includes/css/dist/block-library/style.min.css?ver=*",
  ]
};
```

For a given request, the relevant `Sec-Fetch-Dest` category from `static_dictionaries` is checked as well as the `dynamic_dictionaries` and if the given request matches the URL of a dynamic dictionary or the URLPattern of a static dictionary then the relevant `Use-As-Dictionary` response header is added, including the `match` URLPattern, `match-dest` destination and original request URL as the `id` so that it can be retrieved separately by the compression logic as-needed.

### Compressing the responses

The compression side of things is pretty straightforward. If a request comes in with `Available-Dictionary` and `Dictionary-ID` request headers as well as an `Accept-Encoding` that includes `dcz` (or `dcb` if you are using Brotli) then the response is a candidate for being compressed.

The `Dictionary-ID` *should* be the path to the dictionary that the client has so the workflow basically looks like:

1. Trigger an async fetch of the request from the origin.
1. Fetch the dictionary from the path specified in `Dictionary-ID`.
1. Verify that the SHA-256 hash of the dictionary matches the value in the `Available-Dictionary` request header.
1. Prepare the dictionary for use by Zstandard (or Brotli).
1. Wait for the response from the origin to start.
1. Add the `Content-Encoding: dcz` (or `dcb`) response header.
1. Pipe the response through the compression library using the prepared dictionary.

If any step or check fails, just pass the original response through unmodified using the default encoding.

## Bringing it to production

The basic flow above works great for a proof-of-concept but there are some things that you'll want to implement to improve security, scalability and performance:

### Securing the IDs

Rule #1 in web development is to "never trust the client". Fetching arbitrary URLs from a client-provided `Dictionary-ID` request header could lead to abuse. The easiest way to mitigate the risk is to make sure that the URL provided by the `Dictionary-ID` request header matches the same URLPattern as the request for the static case or that it is one of the configured dynamic dictionary URLs.

It could be secured further by using an ID of sorts (like just the version number) of a given file and generating the actual URLs server-side.

### Chunking the responses

By default, the compression schemes will buffer data until the output buffer is full. This is great for the static file use case where you will get better compression and the whole response streams at the same time.

If you are using early-flushing strategies with your HTML then you are better off forcing the chunks of data to be flushed earlier. In the worker implementation, I flush the first chunk immediately and then buffer a number of chunks before flushing explicitly. This minimizes the added latency while still maintaining good compression.

Different strategies could be used for different content types (and different compression levels as well).

### Caching

There are several places in the flow where caching can be leveraged to reduce the amount of work being done on the edge (the worker implements these as well):

#### Cache the compressed responses

For static resources, it doesn't make sense to re-compress the response every time. You can use the edge cache to store the compressed response using the `Available-Dictionary` and request URL as keys. Before doing any work, the worker checks the cache to see if an appropriate response is already available. Only if one isn't available does it go through the compression steps.

#### Cache the dictionaries

For the dynamic use case, it can be pretty wasteful to fetch and prepare the dictionary on every request. In the case of Cloudflare workers, the V8 isolate that runs the worker is kept alive for lots of requests. The example worker keeps the dictionaries in global memory, indexed by dictionary hash can skip the dictionary fetch and preparation if the dictionary is already available.

## Next steps

It's worth noting that compression dictionary transport is not specific to browsers and the web. It's an HTTP spec that can be used by any app that uses HTTP and it is also available in the Android webview when embedded in apps.

It's still the early days with compression dictionaries and we're still discovering the different ways it can be used and the rough edges. Don't hesitate to reach out though social media, [crbug](crbug.com), [web performance slack](https://webperformance.slack.com/) or whatever else works for you to discuss any questions you might have or ideas you'd like to bounce around.
