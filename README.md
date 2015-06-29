# LeaseWeb API Design Standards

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

## Requests and responses

## Resources

## Pagination and partial Responses

## Search



