# Selfcare API

This is specification of private API used between MALL Pay servers and MALL Pay clients.

There are some rules that are valid throughout whole API.

## Naming conventions

- we use camelCase for field names and object definitions,
- we use plural in resource names,

## Paging

- some resources (stated in documentation) that return collection of objects support pagination.
- on request:
  - query parameter `limit` specifies maximum number of objects in returned collection
  - query parameter `after` specifies last object retrieved in previous request. Its value is usually `id` of last object retrieved in previous call, but this will be stated in documentation. Parameter `after` is used when traversing collection forwards.
  - query parameter `before` specifies first object retrieved in previous request (analogy to `after` parameter), and is used when traversing collection backwards.
  - if `before` and `after` are omitted, beginning of collection is returned, using specified sort order
  - items in collection are always sorted accoridng to attribute which could be passed in after/before parameters. Even if you specify different sorting order, this attribute will be last sorting criterion; if you do not specify sorting order, result collection will be sorted according to this attribute.
- on response:
  - `pagingInfo` object is returned as part of response body with following attributes:
    - _nextPage_ - request to retrieve next page. Either nextPage or previousPage is returned, depending whether you specify `after` or `before` parameter. If you specify neither `before` nor `after` parameter, these attributes will be omitted from response.
    - _prevPage_ - request to retrieve previous page (see `nextPage` attribute description above).
    - _itemsPerPage_ - number of items per page

Example request:

```
curl -X GET https://api.client.mallpay.cz/customer/v1/my/contracts?sort=category&limit=10&after=15
```

Example response pagingInfo:

```javascript
"pagingInfo": {
    "nextPage": "/my/contracts?sort=category&limit=10&after=25"
    "itemsPerPage": 10,
}
```

## Sorting

- some resources (stated in documentation) suppors result sorting. You can specify sorting attributes and order using `sort` request parameter. For ascending order, specify just attribute name; for descending order, add unary - in front of attribute name. You can specify multiple attributes for sorting, separated by comma.
- each resource that supports sorting specifies list of attributes that can be used for sorting.

Examples:

- `/public/fxrates?sort=currencyCode` - get list of FX rates sorted by attribute currencyCode
- `/public/branches?sort=-name` - get list of branches, sorted by attribute name in descending order
- `/banking/accounts?sort=accountType,-accountCurrency,accountName` - get list of accounts, sorted by type (ascending), then by currency descending and then by account name (ascending)

## Filtering

Some resources (stated in documentation) supports results filtering. Such resources have list of filters specified together with possible operations and possible values.

You can specify filtering by passing `filter` attribute. General pattern to specify filter is:

`<filterName>|<operator>|<values>`

- `filterName` - filter name from documentation
- `operator` - operator, specified in resource documentation
- `values` - one or more values for filter. Multiple values are separated by comma

Multiple filters can be specified on each request, separated by semi-colon. They are joined by "AND", so each result item must satisfy all conditions.

All resources whose result object contains `id` have filtering by `id` enabled by default.

**Note**: Some resources (stated in documentation) support filtering by specified embedded child object attributes. Suppose you have the following object structure:

```
{
  "value": {
    "amount": 100,
    "currency": "CZK"
    },
  "someAttribute": "some_attribute_value"
}
```

Then filtering by `amount` can be specified in the filter name as `value.amount`, filtering by currency would be specified as `value.currency` etc. Please se examples below.

### Filtering examples

- get a list of partners with category in (1, 5, 10)

```
GET /general/partners?filter=category|in|1,5,10
```

- get a list of contracts with contractDate in greater than 2016-02-10 and lower or equal to 2016-04-28

```
GET /general/contracts?filter=contractDate|gt|2016-02-10;contractDate|lteq|2016-04-28
```

- get a list of transactions with amount greater then 100 (please see explanation above)

```
GET /general/transactions?filter=value.amount|gt|100
```

### List of operators

| operator  | example value | meaning                                                  |
| --------- | ------------- | -------------------------------------------------------- |
| eq        | 3             | equals 3                                                 |
| gt        | 2             | greater than 2                                           |
| gteq      | 1             | greater than or equals 1                                 |
| icontains | hello         | contains case-insensitive and accent-insensitive "hello" |
| in        | 1,2,3         | value in list (1, 2, 3)                                  |
| isnull    | true          | value is null                                            |
| lt        | 2             | less than 2                                              |
| lteq      | 3             | less than or equals 3                                    |
| abslteq   | 10            | values from -10 to 10                                    |
| absgteq   | 10            | values less than -10 and greater than 10                 |

## API calls limits

When limit is reached, you receive HTTP error 429. To inform you about limits we use following headers:

- `X-Rate-Limit-Limit` - The number of allowed requests in the current period
- `X-Rate-Limit-Remaining` - The number of remaining requests in the current period
- `X-Rate-Limit-Reset` - The number of seconds left in the current period

## Bandwith usage reducing

### Fields attribute

- for all GET resources, you can use optional `fields` query parameter to limit objects' attributes returned in response
- `fields` parameter contains comma-separated list of attributes, that will be present in response; if omitted, all objects' attributes will be returned
- you can specify only top-level attributes in `fields` parameter. This means that when response is an object, you can only specify top-level attributes. When response is an array of objects, only top-level attributes of those objects can be specified.
- if you specify non-existent attribute or atttribute that is not in first level, you will receive HTTP status `400 Bad Request`
- `fields` parameter has no effect on resources that returns plain value or array of plain values.

Examples:

- `/public/branches?fields=id,name,location` - only get list of branches with id, name and location attributes

### GZIP compression

- we support GZIP compression of responses. Client must specify header `Accept-Encoding: gzip` in request in order to use the compression.

## Versioning

We use API version in URL (e.g. `https://api.client.mallpay.cz/customer/v1/my/profile`). Minor changes (see below) that don't break backwards compatibility do NOT increase API version, e.g. they may happen without prior notice and your application should be ready to handle them.

Minor changes include:

- adding new resource
- adding new optional header/URL parameter or optional body attribute to request
- adding new attribute to response body
- adding new error codes and messages, provided that error structure is the same

Every response contains a header `X-API-Version` whose value is in format `X.Y.Z` (1.0.0):

- X - major change, API is not backward compatible
- Y - minor change, API is not backward compatible
- Z - only patch/fix no API changes

## Language

We use English.

## Response encoding

Unless stated otherwise, all responses are sent as `Content-Type: application/json; charset=utf-8`

## HTTP status codes

We use following status codes throughout the API, except for OAuth flow when response codes are prescribed in RFC

- 200 `OK` - request was successful
- 201 `Created` - request was successful and resource was created
- 204 `No content` - we accepted your request but there is nothing to return (e.g. response is empty)
- 400 `Bad Request` - request is missing required parameters
- 401 `Unauthorized` - your API key is wrong or user not authorized (not logged in)
- 403 `Forbidden` - access denied (e.g. user / application is not allowed to use the resource)
- 404 `Not Found` - resource could not be found
- 405 `Method Not Allowed` - specified method is not allowed for resource
- 422 `Unprocessable Entity` - validation errors. Errors are specified in response body (see below)
- 429 `Too Many Requests` - you exceeded the rate limit (see X-Rate-Limit headers)
- 500 `Internal Server Error` - something went wrong on our side
- 503 `Service Unavailable` - there is planned service outage (TODO: should specify response headers with more details on service outage)

## Error handling

Besides HTTP status codes, which are the main indication if something goes wrong, we also use `errors` object to report more details about the errors.

Errors object example:

```javascript
{
    ...
    errors: [
        {
            "code": "ERR_100",
            "message": "Invalid contract number",
            "severity": "ERROR",
            "attribute": "partyAccount.accountNumber",          // optional
            "ticketId": "UAT1:AMS:20160516-091658.450:45e4" // optional
        },
        {
            "code": 352,
            "message": "Insufficiend funds for payment order realization",
            "severity": "WARN"
        },
        {
            "code": 523,
            "message": "This order will trigger currency exchange operation",
            "severity": "INFO"
        }
    ]
}
```

Error object attributes
| attribute name | description |
| --- | --- |
| code | unique error code |
| message | human readable error description (non-localized) |
| severity | error severity (see below) |
| attribute | json path of request attribute that caused the error (optional) |
| ticketId | internal ticket ID, used for error backtracking |

There are 3 levels of error severity:

- ERROR - critical error, execution cannot continue. This MUST be indicated also by appropriate HTTP status code (`422 Unprocessable Entity`)
- WARN - non-critical error, execution can continue but further user interaction is advisable (for request to proceed, you MUST specify this error code in `override` request attribute). This MIGHT be indicated also by appropriate HTTP status code.
- INFO - information only, execution can continue without user interaction.

## Formats

- **date** and **time** uses [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) formatting, e.g.:
  - date is represented as `YYYY-mm-dd`. Timezone is added when necessary.
  - time is represented as `Thh:mm:ss`. Timezone is added when necessary.
  - day of week is represented as number 1..7, with 1 being Monday
  - week no. 1 is the week with the year's first Thursday in it
- **phone numbers** uses international format starting with '+' and including country code
- **numbers format** number format is defined by [JSON standard](http://www.json.org), e.g. decimals are separated by `.`
- **money format** uses [ISO 4217](https://en.wikipedia.org/wiki/ISO_4217) formatting (in minor units, e.g.: 12590 represents 125,90 CZK)

## Documentation principles

- attributes in request/response object are optional, unless stated otherwise (`required` flag under attribute name)
- required attribute in optional object means, that if optional object is specified, it must contain required attribute.
- all values in request/response attributes are just examples, except for enum values - these are the only possible values for given attribute.

## Partial update

We support partial update using PATCH method. Use JSON Merge Patch according to [RFC-7386](https://tools.ietf.org/html/rfc7386).
