# HTTP API 설계 가이드

## 소개

This guide describes a set of HTTP+JSON API design practices, originally
extracted from work on the [Heroku Platform API](https://devcenter.heroku.com/articles/platform-api-reference).
이 가이드에서는 [Heroku 플랫폼 API](https://devcenter.heroku.com/articles/platform-api-reference)를 만들 때의 경험을 바탕으로 HTTP+JSON API를 설계하는 방법에 대해 설명한다.

This guide informs additions to that API and also guides new internal
APIs at Heroku. We hope it’s also of interest to API designers
outside of Heroku.
이 가이드를 통해 API에 포함되어야 하는 내용과 Heroku의 새로운 내부 API에 대해서도 알 수 있다. 
Heroku외부의 API 설계자들에게 이 가이드가 도움이 되길 기대한다.

Our goals here are consistency and focusing on business logic while
avoiding design bikeshedding. We’re looking for _a good, consistent,
well-documented way_ to design APIs, not necessarily _the only/ideal
way_.

We assume you’re familiar with the basics of HTTP+JSON APIs and won’t
cover all of the fundamentals of those in this guide.

We welcome [contributions](CONTRIBUTING.md) to this guide.

## 차례

*  [알맞은 상태 코드를 반환하라](#return-appropriate-status-codes)
*  [전체 리소스를 제공 가능한 곳에서는 전체 리소스를 제공하라](#provide-full-resources-where-available)
*  [요청의 본문에 직렬화된 JSON을 포함시켜라](#accept-serialized-json-in-request-bodies)
*  [리소스의 (UU)ID를 제공하라](#provide-resource-uuids)
*  [표준 타임스탬프를 제공하라](#provide-standard-timestamps)
*  [ISO8601에 정의된 포맷으로 UTC 시간을 사용하라](#use-utc-times-formatted-in-iso8601)
*  [일관된 패스 형태를 사용하라](#use-consistent-path-formats)
*  [패스와 속성은 소문자로 만들어라](#downcase-paths-and-attributes)
*  [외래키 관계는 중첩시켜라](#nest-foreign-key-relations)
*  [편의를 위해 id없는 역참조?(dereferencing)을 지원하라](#support-non-id-dereferencing-for-convenience)
*  [구조적인 에러를 만들어라](#generate-structured-errors)
*  [Etags로 캐시할 수 있도록 지원하라](#support-caching-with-etags)
*  [Request Id로 요청을 추적하라](#trace-requests-with-request-ids)
*  [범위별로 페이지를 나눠라?](#paginate-with-ranges)
*  [사용량의 상태를 보여줘라?](#show-rate-limit-status)
*  [Accepts 헤더에 버전을 부여하라?](#version-with-accepts-header)
*  [경로의 중첩을 최소화하라](#minimize-path-nesting)
*  [기계가 읽을 수 있는 JSON 스키마를 제공하라](#provide-machine-readable-json-schema)
*  [사람이 읽을 수 있는 문서를 제공하라](#provide-human-readable-docs)
*  [실행 가능한 예제를 제공하라](#provide-executable-examples)
*  [안정성을 표현하라?](#describe-stability)
*  [TLS를 요구하라](#require-tls)
*  [기본적으로 예쁜 모양의 JSON을 제공하라](#pretty-print-json-by-default)

### Return appropriate status codes
### 알맞은 상태 코드를 반환하라

Return appropriate HTTP status codes with each response. Successful
responses should be coded according to this guide:

* `200`: Request succeeded for a `GET` calls, and for `DELETE` or
  `PATCH` calls that complete synchronously
* `201`: Request succeeded for a `POST` call that completes
  synchronously
* `202`: Request accepted for a `POST`, `DELETE`, or `PATCH` call that
  will be processed asynchronously
* `206`: Request succeeded on `GET`, but only a partial response
  returned: see [above on ranges](#paginate-with-ranges)

Refer to the [HTTP response code spec](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)
for guidance on status codes for user error and server error cases.

### Provide full resources where available
### 전체 리소스를 제공 가능한 곳에서는 전체 리소스를 제공하라

Provide the full resource representation (i.e. the object with all
attributes) whenever possible in the response. Always provide the full
resource on 200 and 201 responses, including `PUT`/`PATCH` and `DELETE`
requests, e.g.:

```
$ curl -X DELETE \  
  https://service.com/apps/1f9b/domains/0fd4

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
...
{
  "created_at": "2012-01-01T12:00:00Z",
  "hostname": "subdomain.example.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

202 responses will not include the full resource representation,
e.g.:

```
$ curl -X DELETE \  
  https://service.com/apps/1f9b/dynos/05bd

HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
...
{}
```

### Accept serialized JSON in request bodies
### 요청의 본문에 직렬화된 JSON을 포함시켜라

Accept serialized JSON on `PUT`/`PATCH`/`POST` request bodies, either
instead of or in addition to form-encoded data. This creates symmetry
with JSON-serialized response bodies, e.g.:

```
$ curl -X POST https://service.com/apps \
    -H "Content-Type: application/json" \
    -d '{"name": "demoapp"}'

{
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "name": "demoapp",
  "owner": {
    "email": "username@example.com",
    "id": "01234567-89ab-cdef-0123-456789abcdef"
  },
  ...
}
```

### Provide resource (UU)IDs
### 리소스의 (UU)ID를 제공하라

Give each resource an `id` attribute by default. Use UUIDs unless you
have a very good reason not to. Don’t use IDs that won’t be globally
unique across instances of the service or other resources in the
service, especially auto-incrementing IDs.

Render UUIDs in downcased `8-4-4-4-12` format, e.g.:

```
"id": "01234567-89ab-cdef-0123-456789abcdef"
```

### Provide standard timestamps
### 표준 타임스탬프를 제공하라

Provide `created_at` and `updated_at` timestamps for resources by default,
e.g:

```json
{
  ...
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T13:00:00Z",
  ...
}
```

These timestamps may not make sense for some resources, in which case
they can be omitted.

### Use UTC times formatted in ISO8601
### ISO8601에 정의된 포맷으로 UTC 시간을 사용하라

Accept and return times in UTC only. Render times in ISO8601 format,
e.g.:

```
"finished_at": "2012-01-01T12:00:00Z"
```

### Use consistent path formats
### 일관된 패스 형태를 사용하라

#### Resource names

Use the plural version of a resource name unless the resource in question is a singleton within the system (for example, in most systems a given user would only ever have one account). This keeps it consistent in the way you refer to particular resources.

#### Actions

Prefer endpoint layouts that don’t need any special actions for
individual resources. In cases where special actions are needed, place
them under a standard `actions` prefix, to clearly delineate them:

```
/resources/:resource/actions/:action
```

e.g.

```
/runs/{run_id}/actions/stop
```

### Downcase paths and attributes
### 패스와 속성은 소문자로 만들어라

Use downcased and dash-separated path names, for alignment with
hostnames, e.g:

```
service-api.com/users
service-api.com/app-setups
```

Downcase attributes as well, but use underscore separators so that
attribute names can be typed without quotes in JavaScript, e.g.:

```
service_class: "first"
```

### Nest foreign key relations
### 외래키 관계는 중첩시켜라

Serialize foreign key references with a nested object, e.g.:

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  },
  ...
}
```
  
Instead of e.g:

```json
{
  "name": "service-production",
  "owner_id": "5d8201b0...",
  ...
}
```

This approach makes it possible to inline more information about the
related resource without having to change the structure of the response
or introduce more top-level response fields, e.g.:

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0...",
    "name": "Alice",
    "email": "alice@heroku.com"
  },
  ...
}
```

### Support non-id dereferencing for convenience
### 편의를 위해 id없는 역참조?(dereferencing)을 지원하라

In some cases it may be inconvenient for end-users to provide IDs to
identify a resource. For example, a user may think in terms of a Heroku
app name, but that app may be identified by a UUID. In these cases you
may want to accept both an id or name, e.g.:

```
$ curl https://service.com/apps/{app_id_or_name}
$ curl https://service.com/apps/97addcf0-c182
$ curl https://service.com/apps/www-prod
```

Do not accept only names to the exclusion of IDs.

### Generate structured errors
### 구조적인 에러를 만들어라

Generate consistent, structured response bodies on errors. Include a
machine-readable error `id`, a human-readable error `message`, and
optionally a `url` pointing the client to further information about the
error and how to resolve it, e.g.:

```
HTTP/1.1 429 Too Many Requests
```

```json
{
  "id":      "rate_limit",
  "message": "Account reached its API rate limit.",
  "url":     "https://docs.service.com/rate-limits"
}
```

Document your error format and the possible error `id`s that clients may
encounter.

### Support caching with Etags
### Etags로 캐시할 수 있도록 지원하라

Include an `ETag` header in all responses, identifying the specific
version of the returned resource. The user should be able to check for
staleness in their subsequent requests by supplying the value in the
`If-None-Match` header.

### Trace requests with Request-Ids
### Request Id로 요청을 추적하라

Include a `Request-Id` header in each API response, populated with a
UUID value. If both the server and client log these values, it will be
helpful for tracing and debugging requests.

### Paginate with Ranges
### 범위별로 페이지를 나눠라?

Paginate any responses that are liable to produce large amounts of data.
Use `Content-Range` headers to convey pagination requests. Follow the
example of the [Heroku Platform API on Ranges](https://devcenter.heroku.com/articles/platform-api-reference#ranges)
for the details of request and response headers, status codes, limits,
ordering, and page-walking.

### Show rate limit status
### 사용량의 상태를 보여줘라?

Rate limit requests from clients to protect the health of the service
and maintain high service quality for other clients. You can use a
[token bucket algorithm](http://en.wikipedia.org/wiki/Token_bucket) to
quantify request limits.

Return the remaining number of request tokens with each request in the
`RateLimit-Remaining` response header.

### Version with Accepts header
### Accepts 헤더에 버전을 부여하라?

Version the API from the start. Use the `Accepts` header to communicate
the version, along with a custom content type, e.g.:

```
Accept: application/vnd.heroku+json; version=3
```

Prefer not to have a default version, instead requiring clients to
explicitly peg their usage to a specific version.

### Minimize path nesting
### 경로의 중첩을 최소화하라

In data models with nested parent/child resource relationships, paths
may become deeply nested, e.g.:

```
/orgs/{org_id}/apps/{app_id}/dynos/{dyno_id}
```

Limit nesting depth by preferring to locate resources at the root
path. Use nesting to indicate scoped collections. For example, for the
case above where a dyno belongs to an app belongs to an org:

```
/orgs/{org_id}
/orgs/{org_id}/apps
/apps/{app_id}
/apps/{app_id}/dynos
/dynos/{dyno_id}
```

### Provide machine-readable JSON schema
### 기계가 읽을 수 있는 JSON 스키마를 제공하라

Provide a machine-readable schema to exactly specify your API. Use
[prmd](https://github.com/interagent/prmd) to manage your schema, and ensure
it validates with `prmd verify`.

### Provide human-readable docs
### 사람이 읽을 수 있는 문서를 제공하라

Provide human-readable documentation that client developers can use to
understand your API.

If you create a schema with prmd as described above, you can easily
generate Markdown docs for all endpoints with with `prmd doc`.

In addition to endpoint details, provide an API overview with
information about:

* Authentication, including acquiring and using authentication tokens.
* API stability and versioning, including how to select the desired API
  version.
* Common request and response headers.
* Error serialization format.
* Examples of using the API with clients in different languages.

### Provide executable examples
### 실행 가능한 예제를 제공하라

Provide executable examples that users can type directly into their
terminals to see working API calls. To the greatest extent possible,
these examples should be usable verbatim, to minimize the amount of
work a user needs to do to try the API, e.g.:

```
$ export TOKEN=... # acquire from dashboard
$ curl -is https://$TOKEN@service.com/users
```

If you use [prmd](https://github.com/interagent/prmd) to generate Markdown
docs, you will get examples for each endpoint for free.

### Describe stability
### 안정성을 표현하라?

Describe the stability of your API or its various endpoints according to
its maturity and stability, e.g. with prototype/development/production
flags.

See the [Heroku API compatibility policy](https://devcenter.heroku.com/articles/api-compatibility-policy)
for a possible stability and change management approach.

Once your API is declared production-ready and stable, do not make
backwards incompatible changes within that API version. If you need to
make backwards-incompatible changes, create a new API with an
incremented version number.

### Require TLS
### TLS를 요구하라

Require TLS to access the API, without exception. It’s not worth trying
to figure out or explain when it is OK to use TLS and when it’s not.
Just require TLS for everything.

### Pretty-print JSON by default
### 기본적으로 예쁜 모양의 JSON을 제공하라

The first time a user sees your API is likely to be at the command line,
using curl. It’s much easier to understand API responses at the
command-line if they are pretty-printed. For the convenience of these
developers, pretty-print JSON responses, e.g.:

```json
{
  "beta": false,
  "email": "alice@heroku.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "last_login": "2012-01-01T12:00:00Z",
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

Instead of e.g.:

```json
{"beta":false,"email":"alice@heroku.com","id":"01234567-89ab-cdef-0123-456789abcdef","last_login":"2012-01-01T12:00:00Z", "created_at":"2012-01-01T12:00:00Z","updated_at":"2012-01-01T12:00:00Z"}
```

Be sure to include a trailing newline so that the user’s terminal prompt
isn’t obstructed.

For most APIs it will be fine performance-wise to pretty-print responses
all the time. You may consider for performance-sensitive APIs not
pretty-printing certain endpoints (e.g. very high traffic ones) or not
doing it for certain clients (e.g. ones known to be used by headless
programs).
