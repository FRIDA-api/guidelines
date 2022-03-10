# Frida API Guidelines and Best Practies

## Consistency in interfaces eases adoption and lets developers focus on business logic.

Most of the guidance shown below has been adapted from https://integration-services.berkeley.edu/integration-competency-center/api-best-practices.

### Security

#### Require TLS

Require TLS to access the API without exception. It’s not worth trying to figure out or explain when it is alright to use TLS and when it’s not—just require TLS for everything.

#### Authentication

Every API _must_ require at least Basic Authentication. Resources that should be available anonymously _must_ be marked as such.

#### Authorization

Authorization is _not_ handled by the API itself. It's the responsibility of the service layer to ensure the authenticated client has the correct permissions to access the given resource. 

### Contract

#### Status

Clearly communicate in documentation the maturity and stability of your API "contract"—comprised of its various endpoints and formats of its various responses—according to its stage in the "prototype > development > testing > production > deprecation" lifecycle.

#### Versioning

A version number indicates a stable contract your clients can rely on, so include a version "1" from the very start. Don't rely on an unnamed "default" version–always require clients to request one explicitly. Once your API is declared production ready and stable, don't make any backwards incompatible—or "contract-breaking"—changes. When you need to make contract-breaking changes, create a new API with an incremented version number.

While various methods are available for communicating an API's version, the campus convention is to indicate only the major version in the path prefixed with a "v", e.g.:

    https://gateway.api.berkeley.edu/v1/students/10146454

When introducing a new API version, support the previous version during a "cross-over" period to allow clients to transition. Clearly indicate in documentation (and optionally in response headers) when the previous version was deprecated and when you expect to sunset it. Publically provide at most three versions simultaneously: the current production version, a previous deprecated version, and a future prototype or development version.

You may also allow clients to indicate their ability to handle responses from available versions using the HTTP "Accept" header, e.g.:

    Accept: application/json; version=2

### Endpoints

#### Case Convention

Use downcased and dash-separated path names for alignment with hostnames, e.g:

    https://gateway.api.berkeley.edu/users
    https://gateway.api.berkeley.edu/app-setups

#### Resource Names

Use a plural noun to represent a resource (even when specifying a distinct individual via ID) and keep this consistent in the way you refer to resources, e.g.:

    https://gateway.api.berkeley.edu/apps/jupyter-hub/users/10146454

#### Resource References

Reference individual instances of resources with unique IDs. In cases where it's inconvenient for clients to provide IDs you may want to accept either an id or name in the form "/resources/:id\_or\_name", e.g.:

    https://gateway.api.berkeley.edu/apps/97addcf0-c182
    https://gateway.api.berkeley.edu/apps/jupyter-hub

(Do not accept only names to the exclusion of IDs.)

#### Nesting

In resources with nested parent/child relationships, paths may become deeply nested. Depth can be limited by relocating child resources at the root path. For example, in a data model where a compensation belongs to a job belongs to an employee:

replace:

    https://gateway.api.berkeley.edu/employees/10146454/jobs/147/compensations/97a

with

    https://gateway.api.berkeley.edu/employees/10146454/jobs
    https://gateway.api.berkeley.edu/jobs/147/compensations
    https://gateway.api.berkeley.edu/compensations/97a

#### JSON Request Bodies

Accept serialized JSON in **PUT** and **POST** request bodies, either instead of or in addition to form-encoded data.

#### Specialized Actions

Use only HTTP methods (**GET**, **PUT**, **POST**, **DELETE**, etc) to act on resources. In cases where specialized actions must be represented, place them under a standard "actions" node in the form "/resources/:resource/actions/:action" to clearly delineate them, e.g.:

    https://gateway.api.berkeley.edu/employees/10146454/actions/terminate

### Responses

#### Status Codes

Return the appropriate HTTP status code with each response. Successful responses should be coded according to this guide:

*   **200**: Request succeeded for a **GET** call, and for **DELETE** or **PUT** calls that complete synchronously
*   **201**: Request succeeded for a **POST** call that completes synchronously
*   **202**: Request accepted for a **POST**, **DELETE**, or **PUT** call that will be processed asynchronously
*   **206**: Request succeeded on **GET**, but only a partial response is returned

Refer to the [HTTP response code specification(link is external)](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html) for guidance on status codes for client and server error cases.

#### Resource Representation

Provide the full resource representation (i.e. the object with all attributes) whenever possible in the response. Always provide the full resource on **200** and **201** responses (**202** responses will not include the full resource representation).

#### Pretty-print JSON

The first time a developer sees your API response is likely to be at the command line, using curl. It’s much easier to understand responses if they are pretty-printed. For that convenience, pretty-print JSON responses by default, e.g.:

    {
      "beta": false,
      "email": "steve@berkeley.edu",
      "id": "01234567-89ab-cdef-0123-456789abcdef",
      "lastLogin": "2012-01-01T12:00:00Z",
      "createdDateTime": "2012-01-01T12:00:00Z",
      "updatedDateTime": "2012-01-01T12:00:00Z"
    }

(Be sure to include a trailing newline so that the user’s terminal prompt isn’t obstructed.)

For most APIs pretty-printing by default won't overly degrade performance. For performance-sensitive APIs consider not pretty-printing certain responses.

#### Parent/Child Representation

Gather child attributes together within a nested object, e.g.:

replace

    {
      "name": "service-production",
      "ownerId": "5d8201b0",
      ...
    }

with

    {
      "name": "service-production",
      "owner": {
        "id": "5d8201b0",
        ...
      },
      ...
    }

This approach makes it possible to include more information about the child without having to alter the structure of the response or introduce more top-level fields, e.g.:

    {
      "name": "service-production",
      "owner": {
        "id": "5d8201b0",
        "name": "russ",
        "email": "russ@berkeley.edu"
      },
      ...
    }

#### Timestamps

Accept and return time attributes in UTC ISO-8601 format only, e.g.:

    "finishedDateTime": "2012-01-01T12:00:00Z"

For changeable resources, provide "created" and "updated" timestamps by default, e.g:

    {
      ...
      "createdDateTime": "2012-01-01T12:00:00Z",
      "updatedDateTime": "2012-01-01T13:00:00Z",
      ...
    }

####   
Paginatation

Paginate any responses that are liable to produce large amounts of data. Several pagination methods exist, so just be consistent.

#### Rate Limit Status

Rate-limit requests from clients to protect the health of your service and maintain high service quality for all of your clients. Include the remaining number of request units as part of each response in a "RateLimit-Remaining" header.

#### Request UUIDs

Include a "Request-Id" header populated with a service-generated UUID value in each response. If both the server and client log these values, it will be helpful for tracing and debugging requests. Render UUIDs in downcased 8-4-4-4-12 format, e.g.:

    "Request-Id": "01234567-89ab-cdef-0123-456789abcdef"

#### Caching Etags

Include an ETag header in all responses, identifying the specific version of the returned resource. The user should be able to check for staleness in their subsequent requests by supplying the value in the If-None-Match header.

#### Structured Errors

Generate consistent, structured response payloads for errors. Include a machine-readable error id, a human-readable error message, and optionally a URI pointing the client to further information about the error and how to resolve it, e.g.:

    HTTP/1.1 429 Too Many Requests

    {
      "id": "rateLimit",
      "message": "Account has reached its API rate limit.",
      "url": "https://docs.berkeley.edu/rate-limits"
    }

Include your error response format and all error codes that clients may encounter in your documentation.

### Documentation

#### Overview

In addition to endpoint and schema specifics, provide human-readable information about:

*   Authentication, including acquiring and using authentication tokens
*   API state and version, including how to call a specific version
*   Expected request and response headers, especially custom ones
*   Error response format
*   Examples of using the API with clients in different languages

#### Endpoints

List and describe the action of all valid HTTP methods for each endpoint. Include all valid parameters, their acceptable values, whether they are required, and their default values, if any.

#### Responses

Provide machine-readable schema exactly specifying each response from each endpoint in all provided formats (e.g., JSON and/or XML). Show examples of each response, or provide a mock service.

#### Interaction

Provide executable examples that users can type directly into their terminals to see working API calls. To the greatest extent possible, these examples should be usable verbatim, to minimize the amount of work a user needs to do to try the API.

#### OpenAPI

Consider providing an "[OpenAPI(link is external)](https://www.openapis.org)" (previously "Swagger") document for your API. An OpenAPI definition can be used to:

*   generate interactive documentation (such as on API Central) to display the API
*   generate server and client code in various programming languages
*   configure testing tools

and many other use cases.
