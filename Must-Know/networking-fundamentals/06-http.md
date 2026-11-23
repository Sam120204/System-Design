# HTTP and HTTPS

HTTP defines requests and responses. A request carries a method, a target, headers, and an optional body. A response carries a status code, headers, and an optional body. HTTPS protects HTTP with TLS, which encrypts traffic, checks its integrity, and normally authenticates the server using a certificate.

## Read a request and response

[![Annotated HTTP request and response showing the method, request headers, status code, response headers, and body](https://www.hellointerview.com/_next/static/media/H1gTar0isNJz.1pa8cxsbnoo8a.svg)](images/http-request-and-response.png)

```http
GET /products/42 HTTP/1.1
Host: shop.example
Accept: application/json
If-None-Match: "product-42-v7"
```

```http
HTTP/1.1 304 Not Modified
ETag: "product-42-v7"
Cache-Control: max-age=60
```

The client can reuse its cached representation after receiving the `304`, which has no body. `Cache-Control` defines freshness and reuse policy, `ETag` identifies a version of a representation, and `Vary` identifies request headers that affect cache matching. For personalized content, explicitly choose cache directives appropriate to the sensitivity of the response and check the CDN cache key. [HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)

## Methods and response codes

- **GET** retrieves a representation and has safe semantics, meaning the client asks for a read. **PUT** replaces or creates the state of a known resource as defined by the endpoint. **DELETE** removes the target resource's association with its URI. PUT and DELETE have idempotent semantics, so a repeated identical request has the same intended effect as a single request even when the returned status changes.
- **POST** asks the target to process an input and often creates a resource. Supply an idempotency mechanism when a caller needs to retry a write whose outcome may be unknown.
- **Codes** communicate the outcome. `201` indicates creation, `202` indicates accepted processing, `401` means valid authentication credentials are required, `403` indicates refusal to fulfill the request, `429` signals rate limiting, and `503` signals temporary unavailability. `Retry-After` can communicate when the server expects a retry to be appropriate.

An HTTP connection can be reused across requests. Application session state, such as login information, is typically represented by a cookie, token, or server-side record and can outlive a connection. [HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)

## Compare HTTP versions

| Version | Typical transport | Request behavior |
| --- | --- | --- |
| HTTP/1.1 | TCP, with TLS for HTTPS | Persistent connections permit reuse. Responses to pipelined requests are sent in order |
| HTTP/2 | TCP, typically with TLS in browser deployments | Binary frames, header compression, and multiplexed streams. TCP loss can delay multiple streams |
| HTTP/3 | QUIC over UDP | HTTP streams use QUIC's stream-level delivery while sharing connection and network capacity |

A browser might use HTTP/3 to a CDN while the CDN uses HTTP/2 to the origin. Choose the protocol per hop based on support and operational requirements. [HTTP/2 specification](https://www.rfc-editor.org/rfc/rfc9113.html) · [QUIC](05-quic.md)
