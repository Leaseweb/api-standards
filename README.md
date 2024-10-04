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
  - [Naming convention](#naming-convention) 
  - [Plural nouns](#plural-nouns)
  - [Use CRUD](#use-crud)
  - [Non-CRUD operations](#non-crud-operations)
  - [Leave complexity behind the query string](#leave-complexity-behind-the-query-string)
  - [Responses that don't involve a resource](#responses-that-don-t-involve-a-resource)
+ [Pagination](#pagination)
  - [Use offset and limit](#use-offset-and-limit)
  - [Attach metadata](#attach-metadata)
  - [Use defaults](#use-defaults)
+ [Metrics](#metrics)
  - [Granularity](#granularity)
  - [Aggregation](#aggregation)
+ [Partial Responses and Filters](#partial-responses-and-filters)
  - [Subsets of Fields](#subsets-of-fields)
  - [Filtering](#filtering)
+ [Search](#search)
+ [Asynchronous Calls](#asynchronous-calls)

This guide describes a set of API Design standards used at [LeaseWeb](https://www.leaseweb.com).
When designing a new API, it's important to respect the HTTP interaction patterns
and resource models described below.

In case you need to defer from these standards or if conflicts are found, please contact the Developer Platform.



## Foundation

### Require TLS

Require TLS to access the API, without exception. Ideally, simply reject any non-TLS
requests by not responding to requests for HTTP or port 80 to avoid any insecure data
exchange. Respond with a `403 Forbidden` if this is not possible for your environment.


Redirects are discouraged since they allow sloppy/bad client behavior without providing
any clear gain. Clients that rely on redirects double up on server traffic and render TLS
useless since sensitive data will already have been exposed during the first call.

### Versioning

APIs are subject to version control. Versioning your API is important as it helps developers
and existing clients to slowly transition from a version to another with enough time to plan
and adapt on changes.

Keep your API version connected to interfaces changes rather than implementations. If you fix
a bug on version 1, is just easier to leave existing clients on version 1 rather than ask all of them
to switch to 1.1 because you fixed something.
Should you change the interface, perhaps returning a completely different type in a response or
having different mandatory parameters, then is a good time to inform developers that they need to
switch soon or later to version 2.

The version number of an API should appear in its URI as `/vN` with the major version (`N`) as prefix.

#### Example

`/v1/bareMetals`

### Trace requests with Correlation-Ids

Each API response through the API Gateway will include a `APIGW-CORRELATION-ID` header
populated with a UUID value. Both server and client can log these values, which will be helpful
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
* `204 No Content`: Request successfully processed a `POST` or `DELETE` call, but is not returning any content

Use the following HTTP status codes for errors:

* `400 Bad Request`: Request failed because client submitted invalid data and needs to check his data before submitting again
* `401 Unauthorized`: Request failed because user is not authenticated
* `403 Forbidden`: Server understood the request, but refuses to fulfill it (might be due to the status of the resource, see the error message for specifics)
* `404 Not Found`: Request failed because the resource does not exist or the resource doesn't belong to the user (e.g. for a `GET`, `PUT` or `DELETE` the specified resource doesn't exist)
* `405 Method Not Allowed`: Request failed because the requested method is not allowed
* `500 Internal Server Error`: Request failed on server side, user should check status site or report the issue (preferably we track 500 errors and get notified automatically)
* `503 Service Unavailable`: API is unavailable, check status site for details


### Provide full resources

Provide the full resource representation in the response. Always provide the full resource on `200` and `201` responses, except
for `DELETE` requests.

In case of asynchronous calls (`POST/PUT/DELETE`) you should use `202 Accepted`. `202` Responses will not include the full resource 
representation. You can provide a task ID which can be queried, or create a temporary resource until the asynchronous call has 
finished and the resource is created. See the [Asynchronous Calls](#asynchronous-calls) for more information.

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
    "errorCode" : "APP00800",
    "errorMessage" : "The connection with the DB cannot be established.",
    "correlationId" : "550e8400-e29b-41d4-a716-446655440000",
    "userMessage" : "Cannot handle your request at the moment. Please try again later.",
    "reference" : "http://developer.leaseweb.com/errors/APP00800"
}
```

### Form validation
You may need to return several error messages when dealing with form data. In this case use additional response field “errorDetails”
with explicit information about all errors.

#### Example Response
```json
HTTP Status: 400 Bad Request

{
    "errorCode" : "APP00900",
    "errorMessage" : "Validation failed.",
    "correlationId" : "550e8400-e29b-41d4-a716-446655440000",
    "userMessage" : "Your data contains errors, please check details.",
    "reference" : "http://developer.leaseweb.com/errors/APP00900",
    "errorDetails" : {
        "firstName" : ["Name cannot be empty", "Name must be unique"],
        "country" : ["Country cannot be empty"]
    }
}
```


## Resources

### Naming convention
Use the `camelCase` convention for naming parameters, endpoints and query string keys. for example:

| Incorrect | Correct |
| ---  | --- |
| `/v1/bareMetal-servers` | `/v1/bareMetalServers` |
| `/v1/public_cloud` | `/v1/publicCloud` |
| `/v1/networkequipments` | `/v1/networkEquipments` |
| `/v1/bareMetalServers?sort-by=location` | `/v1/bareMetalServers?sortBy=location` |


### Plural nouns
Use the plural nounce for collection resource names. It makes it easy and predictable for developers writing an API using consistent names and distinguishes
between collections and singletons.

### Use CRUD
Use the standard HTTP verbs `GET`,`POST`,`PUT` and `DELETE` to operate on collections or singletons.

| Verb | Usage | Idempotent | |
| ---  | ---   | ---        |	---	 |
| GET   | Read  | X         | Reads a collection or singleton	|
| POST  | Create|           | Creates a singleton |
| PUT   | Update| X         | Updates a collection (bulk) or a singleton. Partial updates are allowed |
| DELETE| Delete| X         | Deletes a complete collection or a singleton |


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


## Pagination
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

## Metrics
API design for metrics/usage across all products.

### Granularity
Data granularity is a measure of the level of detail in a data structure. In time-series data, for example, the granularity of measurement might be based on intervals of years, months, weeks, days, or hours. Leaseweb API supports:

- `5MIN`
- `HOUR`
- `DAY`
- `WEEK` (Calendar week, starting on Monday according to ISO8601)
- `MONTH`
- `YEAR`


### Aggregation
Aggregation means: instead of getting metrics or usage for a single entity, we should be able to give combined metrics for multiple entities (for example 20 at once). Leaseweb API supports:

- `SUM`
- `AVG`

#### Example Request
`GET /v1/instances/granularity=DAY&aggregation=SUM`


## Partial Responses and Filters

### Subsets of fields
Sometimes you don't need an entire object, you just need a part of it. You can choose which fields to return by specifying a `fields` query parameter.

#### Example Request
`GET /v1/domains?fields=id,name`

This request will return a collection of domains containing only the resource id and name.

### Filtering

You can use a filtering expression to retrieve specific items. Add a `filter` field to querystring with the expression you need.
We support the following expressions:

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

## Asynchronous calls
Sometimes a request cannot return the full resource result immediately. To handle this type of requests you can use a job queue or other sort of job management internally and use API requests to manage it.
Jobs should be a normal resource in your API, but without the normal CRUD operations. Check the [Resources](#resources) for more information about listing job collections or singletons.

### Create a job
Your method behaves as a normal `POST`/`PUT`/`DELETE` endpoint, but the response is a `202 Accepted`. The header `Location` has a URL to get the information of the job. In the response body the information of the job is provided.

#### Example Response
```json
HTTP/1.1 202 Accepted
Location: https://api.leaseweb.com/v1/jobs/237daad0-2aed-4260-b0e4-488d9cd55607
Retry-After: 30

{
  "id": "237daad0-2aed-4260-b0e4-488d9cd55607",
  "name": "virtualServer.provision",
  "status": "PENDING",
  "createdAt": "2016-12-31T01:02:03+00:00",
  "eta": "2016-12-31T01:02:03+00:00"
}
```

### Status of a job
To retrieve the current status of a job use a `GET` request to the jobs endpoint together with the ID of the job. The response body contains information about the job, which must include the ID, the name that describes the job, the current status and the dates when the job was created and started in ISO-8601 format. Optional the response can also contain an estimated end date for the job in ISO-8601 format. Also optional is including the header `Retry-after` that contains an estimated time in seconds to retry the request for the job status.

#### Example Response
```json
HTTP/1.1 200 OK
Retry-After: 30

{
  "id": "237daad0-2aed-4260-b0e4-488d9cd55607",
  "name": "virtualServer.provision",
  "status": "STARTED",
  "createdAt": "2016-12-31T01:02:03+00:00",
  "startedAt": "2016-12-31T01:02:04+00:00",
  "eta": "2016-12-31T01:02:03+00:00"
}
```

### Completed job
When the job is completed the response is a `303 See Other` together with a `Location` header with the URL to get the information of the resulting resource. The response body contains all the information about the job and the date when job was completed in ISO-8601 format.

#### Example Response
```json
HTTP/1.1 303 See Other
Location: https://api.leaseweb.com/v1/virtualServers/52f606e0-61d3-4355-a191-56f5ab8b26f8

{
  "id": "237daad0-2aed-4260-b0e4-488d9cd55607",
  "name": "virtualServer.provision",
  "status": "COMPLETED",
  "createdAt": "2016-12-31T01:02:03+00:00",
  "startedAt": "2016-12-31T01:03:05+00:00",
  "completedAt": "2016-12-31T01:04:25+00:00"
}
```

If the job doesn't refer to a resource the response is a `200 OK` without the `Location` header, and the response body contains the all information about the job.

### Cancel a job
To cancel an uncompleted job you can use a `DELETE` request to the job endpoint together with the ID of the job.

If a job was successfully canceled the response is a `200 OK` with a response body containing the information about the job.

#### Example Response
```json
HTTP/1.1 200 OK

{
  "id": "237daad0-2aed-4260-b0e4-488d9cd55607",
  "name": "virtualServer.provision",
  "status": "CANCELED",
  "createdAt": "2016-12-31T01:02:03+00:00"
}
```

Sometimes the job might be in a state that it cannot be canceled. In this case the response is a `403 Forbidden`.

### Purged jobs
Purging can be done automatically using internal methods or not done at all. A request to an already purged job will return a `404 Not Found` response.
