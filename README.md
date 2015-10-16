# LeaseWeb API Design Standards

+ [Foundation](#foundation)
  - [Require TLS](#require-tls)
  - [Versioning](#versioning)
  - [Trace requests with Correlation-Ids](#trace-requests-with-correlation-ids)
+ [Requests and responses](#requests-and-responses)
  - [HTTP Status codes](#http-status-codes)
  - [Provide full resources](#provide-full-resources)
  - [Error messages](#error-messages)
  - [Form validation](#form-validation)
+ [Resources](#resources)
  - [Plural nouns](#plural-nouns)
  - [Use CRUD](#use-crud)
  - [Non-CRUD operations](#non-crud-operations)
  - [Leave complexity behind the query string](#leave-complexity-behind-the-query-string)
  - [Responses that don't involve a resource](#responses-that-don-t-involve-a-resource)
+ [Pagination](#pagination)
  - [Use offset and limit](#use-offset-and-limit)
  - [Attach metadata](#attach-metadata)
  - [Use defaults](#use-defaults)
+ [Partial Responses and Filters](#partial-responses-and-filters)
  - [Subsets of Fields](#subsets-of-fields)
  - [Filtering](#filtering)
+ [Search](#search)

This guide describes a set of API Design standards used at [LeaseWeb](www.leaseweb.com).
When designing a new API, it's important to respect the HTTP interaction patterns
and resource models described below.

In case you need to defer from these standards or if conflicts are found, please contact the Developer Platform.


##Foundation

### Require TLS

Require TLS to access the API, without exception. Ideally, simply reject any non-TLS 
requests by not responding to requests for http or port 80 to avoid any insecure data 
exchange. Respond with a `403 Forbidden` if this is not possible for your environment.


Redirects are discouraged since they allow sloppy/bad client behavior without providing 
any clear gain. Clients that rely on redirects double up on server traffic and render TLS 
useless since sensitive data will already have been exposed during the first call.

### Versioning

APIs are subject to version control. Versioning your API is important as it helps developers 
and existing clients to slowly transition from a version two another with enough time to plan 
and adapt on changes.

Keep your API version connected to interfaces changes rather than implementations. If you fix 
a bug on version 1, is just easier to leave existing clients on version 1 rather than ask all of them 
to switch to 1.1 because you fixed something.
Should you change the interface, perhaps returning a completely different type in a response or 
having different mandatory parameters, then is good time to inform developers that they need to 
switch soon or later to version 2.

The version number of an API should appear in its URI as `/vN` with the major version (`N`) as prefix. 

#### Example

`/v1/bareMetals`

### Trace requests with Correlation-Ids

Each API response through the API Gateway will include a `APIGW-CORRELATION-ID` header 
populated with a UUID value. Both the server and client can log these values, which will be helpful 
for tracing and debugging requests.


If you make subsequent calls to other APIs due to an API request, it is advised to add the 
`APIGW-CORRELATION-ID` header to your subsequent call. This way it is possible to trace an 
API call from start to end throughout our application landscape.

When the API Gateway isn’t receiving a `APIGW-CORRELATION-ID` it will add the header to the call.
 


## Requests and responses

### HTTP Status codes

Successful responses should be coded according to this guide:

* `200 OK`: Request succeeded for a `GET` call, for `PUT` or `DELETE` call that completed synchronously
* `201 Created`: Request succeeded for a `POST` call that completed synchronously
* `202 Accepted`: Request accepted for a `POST`, `PUT` or `DELETE` call that will be processed asynchronously 
* `204 No Content`: Request successfully processed a `DELETE` call, but is not returning any content 

Use the following HTTP status codes for errors:

* `400 Bad Request`: Request failed because client submitted invalid data and needs to check his data before submitting again 
* `401 Unauthorized`: Request failed because user is not authenticated
* `403 Forbidden`: Request failed because user does not have authorization to access the resource
* `404 Not Found`: Request failed because the resource does not exists
* `405 Method Not Allowed`: Request failed because the requested method is not allowed
* `500 Internal Server Error`: Request failed on server side, user should check status site or report the issue (preferably we track 500 errors and get notified automatically)
* `503 Service Unavailable`: API is unavailable, check status site for details


### Provide full resources

Provide the full resource representation in the response. Always provide the full resource on `200` and `201` responses, except 
for `DELETE` requests.

In case of asynchronous calls (`POST/PUT/DELETE`) you should use `202 Accepted`. `202` Responses will not include the full resource 
representation. You can provide a task ID which can be queried, or create a temporary resource until the asynchronous calls has 
finished and the resource is created.

*NOTE: see the examples for single resources and collections. In a single resource there is no need for a root-identifier.*


### Error messages
Always provide an error code in your responses, so that applications can always understand what is specifically wrong. 
Additionally, provide 2 different error messages: one very technical for developers and one that may be directly shown to an 
end user. Developers will like this approach since it requires less code handling for them.

Optionally add a link to a page for further explanation on the error.

#### Example Response
```json
HTTP Status: 500 Internal Server Error

{
    "errorCode"    : "APP00800",
    "errorMessage" : "The connection with the DB cannot be established.",
    "userMessage"  : "Cannot handle your request at the moment. Please try again later.",
    "reference"    : "http://developer.leaseweb.com/errors/APP00800"
}
```

### Form validation
You may need to return several error messages when dealing with form data. In this case use additional response field “errorDetails” 
with explicit information about all errors.

#### Example Response
```json
HTTP Status: 400 Bad Request

{
    "errorCode"      : "APP00900",
    "errorMessage" : "Validation failed.", 
    "userMessage"  : "Your data contain errors, please check details.",
    "reference"       : "http://developer.leaseweb.com/errors/APP00900",
    "errorDetails"    : {
        "firstName"    : ["Name cannot be empty", "Name must be unique"],
        "country" : ["Country cannot be empty"]
    }
}
```
 

## Resources
 
### Plural nouns
Use the plural nounce for collection resource names. It makes it easy and predictable for developers writing an API using consistent names and distinguishes 
between collections and singletons.

### Use CRUD
Use the standard HTTP verbs `GET`,`POST`,`PUT` and `DELETE` to operate on collections or singeltons.

| Verb | Usage | Idempotent | |
| ---  | ---   | ---        |	---	 |
| GET   | Read  | X          | Reads a collection or singleton	|
| POST  | Create|            | Creates a singleton |
| PUT   | Update| X           | Updates a collection (bulk) or a singleton. Partial updates are allowed |
| DELETE| Delete|            | Deletes a complete collection or a singleton |

### Non-CRUD operations


### Leave complexity behind the query string
If you need to filter a specific resource, make use of the query string.

#### Example Request
`GET /v1/bareMetalServers?location=ams-01&state=running`

### Responses that don't involve a resource
When performing actions like:
+ Calculate
+ Translate
+ Convert
you are not operating on a resource, therefore the best would be to use a verb and not a noun for your endpoint.

#### Example Request
`GET /convert?from=EUR&to=USD&amount=100`

Make it clear in your API documentation that these *non-resource* scenarios are different.


## Pagination and partial Responses
Pagination is an essential part of any API exposing a collection of results.
Developers must always be aware that a response has more results than requested and 
should have an easy way to request those additional resources.

Moreover, simple calls that require only partial data, like for instance just a property of an 
object instead of the whole attributes list, must be accommodated easily. For this reason, 
clients may just pass a list of the fields they really require, keeping your API results concise 
and fast.

### Use `offset` and `limit`
Use `offset` and `limit` to request a range for a response. It is very common and not misleading. 

#### Example Request
`GET /v1/domains?limit=25&offset=50`

This request will return a collection of 25 domains starting from the 50th.

### Attach metadata
Always inform the developer that the response is paginated with a metadata attribute in your response, 
specifying the total number of items in the collection, the limit and the offset.

#### Example Response
```json
{ 
	"domains": [
		{ "id": "leaseweb.com" },
		{ "id": "leaseweb.net" }
	],
	"_metadata": {
		"totalCount": 132,
		"limit": 2,
		"offset": 0
	}
}
```

### Use defaults
By rule of thumb, you may define a default limit of 25 and offset of 0. Of course, if your application serves large amount of data per request, 
you may reduce the default limit and vice versa.

## Partial Responses and Filters

### Subsets of fields
Sometimes you don't need an entire object, you just need a part of it. You can choose which fields to return by specifying a `fields` query parameter.

#### Example Request
`GET /v1/domains?fields=id,name`

This request will return a collection of domains containing only the resource id and name.

### Filtering

You can use a filtering expression to retrieve specific items. Add a `filter` field to querystring with the expression you need.
We support the follwing expressions:

#### Equals

The following request will return a collection of domains where the domainName equals "leaseweb.com":

`GET /v1/domains?filter={ "domainName" : "leaseweb.com" }`

or 

`GET /v1/domains?filter={ "domainName" : { "$eq" : "leaseweb.com" } }`

#### Not equals

The following request will return a colletion of domains where the domainName is not equal to "leaseweb.com":

`GET /v1/domains?filter={ "domainName" : { "$ne" : "leaseweb.com" } }`

#### Comparison

The following request will return a collection of bareMetalServers which have a `id` value greater than 23:

`GET /v1/bareMetalServers?filter={ "id" : { "$gt" : 23 } }`

You can also use `$lt` (less-than), `$gte` (greater-than-or-equal-to)  and `$lte` (less-than-or-equal-to) comparisons.

#### Boolean expressions

It is possible to combine expressions to get items. The following request will return a collection of bareMetalServers which have an `id` greater than 23 and `reference` equals "TEST":

`GET /v1/bareMetalServers?filter={ "$and": [{ "id" : { "$gt" : 23 } }, { "reference" : "TEST"} ] }`

You can also use `$or` (either of the conditions)  and `$nor` (neither of the conditions)  operators.

## Search
 
### Scoped Search
When the search scope is narrower and on a specific resource, you can use the “q” parameter to provide the search query.

#### Example Request
`GET /v1/domains/leaseweb.com/dnsRecords?q=amsterdam`

This will list all dnsRecords within the domain resource identified by `leaseweb.com` that contains the text `amsterdam`.

*Note: search is not filtering*


