# Data service generation for bal persist: HTTP

- Author: @niveathika
- Reviewers: 
- Created: 2026-10-07
- Updated: 2026-10-26
- Issue: [#1513](https://github.com/ballerina-platform/ballerina-spec/issues/1513)
- Status: Draft

## Summary

This proposal defines how the protocol-neutral CRUD shape in
[Data service generation for bal persist](https://github.com/ballerina-platform/ballerina-spec/issues/1503)
is rendered as an `http:Service` when `options.dataservice.protocol = "http"`.

Each exposed entity becomes a collection resource and a key resource under the base path, with
a resource per relation under the key. The seven operations map to `GET`, `POST`, `PATCH` and
`DELETE`, the query options map to query parameters, results are JSON, and the error categories
map to status codes.

Each case marked ✅ has been compiled and run. Unless its text names another setup, it ran on
Ballerina 2201.13.3 with `ballerina/http` 2.17.3 and a generated persist in-memory client, and
the implementation's tests recheck it on H2 with Ballerina 2201.14.0. The implementation
targets Ballerina 2201.14.0 and the SQL datastores.

The examples use the `Employee` and `Department` entities of #1503.

## Goals

- Define the URL, method, request body, response body and status code of every operation.
- Use the persist-generated types as the request and response bodies, except where a field's
  JSON form must differ: date and time values and `byte[]`.
- Use plain REST conventions: the key in the path, `201` with `Location` on create, `PATCH`
  for a partial update.

## Non-goals

- Row-level auth and nested writes. They follow the roadmap in #1503.
- `PUT` as a full replace. The persist client has no replace operation.
- Compatibility with the wire format of WSO2 MI data services.

## Design

### Listener and service

`dataservice.bal` declares the persist client, one listener and one service in the default
module:

```ballerina
final Client dataserviceClient = check new ();
listener http:Listener dataserviceListener = new (dataservicePort, dataserviceListenerConfig);

service /api on dataserviceListener {
    // one group of resources per exposed entity
}
```

- **Base path:** the service's base path is `options.dataservice.basePath`, `/api` by default.

### Operations

For an entity `Employee` whose persist resource name is `employees`, with key `id`:

| Operation | Action | Request | Success response |
|---|---|---|---|
| Get many | `read` | `GET /api/employees` with the [query parameters](#query-parameters) | `200`, `{"items": [Employee, ...], "hasMore": false}` |
| Get one | `read` | `GET /api/employees/{id}`, optionally with `fields` | `200`, `Employee` |
| Get related, to-one | `read` | `GET /api/employees/{id}/department`, optionally with `fields` | `200`, `Department` |
| Get related, to-many | `read` | `GET /api/departments/{id}/employees` with the query parameters | `200`, `{"items": [Employee, ...], "hasMore": false}` |
| Create one | `create` | `POST /api/employees` with an `EmployeeInsert` object | `201`, `Location: /api/employees/{id}`, the created `Employee` |
| Create many | `create` | `POST /api/employees` with an array of `EmployeeInsert` | `201`, `{"items": [Employee, ...], "hasMore": false}` |
| Patch one | `update` | `PATCH /api/employees/{id}` with an `EmployeeUpdate` object | `200`, the updated `Employee`; `204` when the entity does not allow `read` |
| Delete one | `delete` | `DELETE /api/employees/{id}` | `200`, the deleted `Employee`; `204` when the entity does not allow `read` |

- **Create one and create many share one route.** The payload parameter is typed
  `EmployeeInsert|EmployeeInsert[]`; `ballerina/http` binds a JSON object to the first type and
  an array to the second. ✅
- **Creates are atomic.** Create one and create many run the insert and the read-back in one
  transaction, so a batch with one failing record creates none. ✅ On H2 with
  `ballerinax/persist.sql` 1.7.4, a batch with a duplicate key in the middle saved the other
  records without a transaction, and saved none inside a Ballerina `transaction`.
- **Create many has no `Location` header**, because it creates several resources.
- **Create many takes at most `dataserviceMaxPageSize` records.** A larger array is answered
  with `400` and `INVALID_INPUT`.
- **Patch is partial.** Fields left out of the body are not changed. A field set to `null` is
  set to `null` when the field is nullable.
- **Delete returns the deleted record**, as the persist client does, when the entity allows
  `read`. A client that does not need it can ignore the body.
- **Without `read`,** patch and delete answer `204 No Content` with no body.
- **An empty array creates nothing.** `POST` with `[]` is answered with `201` and
  `{"items": [], "hasMore": false}`, without calling the persist client.
- **An empty patch changes nothing.** `PATCH` with `{}` is answered as get one when the entity
  allows `read`, and with `204` without reading otherwise. Neither calls the persist client's
  update, which would have no columns to set.
- **Key fields cannot be patched.** `EmployeeUpdate` is a closed record without the key fields,
  so a body that sets one is answered with `400` by `ballerina/http`.
- **Get many without `sort`** returns records ordered by key, as #1503 defines.

#### Disabled operations

An operation whose action is not enabled is not generated, and `ballerina/http` answers a
request for it on its own:

| Situation | Response |
|---|---|
| A method that exists on no resource of the path, for example `PATCH` on a read-only entity | `405 Method Not Allowed` ✅ |
| A path that matches no resource, for example an entity that is not exposed | `404 Not Found` |

### Keys

- **Single key:** one path segment, `/employees/{id}`, typed as the key field's type.
- **Composite key:** one segment per key field, in declaration order, for example
  `/assignments/{employeeId}/{projectCode}`. ✅ There is no common convention for composite keys
  (see [Alternatives](#alternatives)). Positional segments are the conventional REST form for a
  single key, extend that form without new syntax, and match the persist client's resource
  paths.
- **Key types:** persist allows only `int`, `string`, `float`, `decimal` and `boolean` key
  fields, and each binds as a path parameter. A segment that does not convert to the key's type
  is answered with `400` by `ballerina/http`. ✅ A `float` key of `NaN` or `Infinity` is answered
  with `400` and `INVALID_INPUT`, with the message #1503 gives.
- **Key text:** in `Location` and in error messages, a key value is written with Ballerina's
  `toString`, so `1.5` stays `1.5` and a `decimal` keeps its scale. In `Location`, each value
  is then percent-encoded as a path segment.

### Types and JSON

The resources use the persist-generated types directly: `Employee` in responses,
`EmployeeInsert` and `EmployeeUpdate` in requests.

| Ballerina type | JSON |
|---|---|
| `int`, `float`, `decimal` | number |
| `string`, `boolean` | string, boolean |
| `T?` | the value or `null` |
| enum | string |
| `byte[]` | a base64 string (RFC 4648, standard alphabet, padded) |
| `time:Date` | an ISO 8601 date, `2024-01-02` |
| `time:TimeOfDay` | an ISO 8601 time, `10:15:30` or `10:15:30.5` |
| `time:Civil` | an ISO 8601 date-time without an offset, `2024-01-02T10:15:30` |
| `time:Utc` | an ISO 8601 date-time in UTC, `2024-01-02T10:15:30Z`. In a request, any offset is accepted and converted to UTC |

Ballerina's own JSON form of these types does not suit a client: `time:Utc` is a tuple and
becomes an array such as `[1704189330, 0.5]`, the other `time:*` types become records, and
`byte[]` becomes an array of numbers.

#### Wire types

An entity gets wire types when it has a `time:*` or `byte[]` field, which the wire types hold as
`string`, or a field with a [field rule](#field-rules). Each wire type is generated only when an
operation that uses it is generated. The other entities use the persist types as they are.

| Persist type | Wire type |
|---|---|
| `Employee` | `EmployeePayload` |
| `EmployeeInsert` | `EmployeeInsertPayload` |
| `EmployeeUpdate` | `EmployeeUpdatePayload` |

```ballerina
public type EmployeePayload record {|
    readonly int id;
    string name;
    string? email;
    string hiredOn;
    Status status;
    int departmentId;
|};
```

The `<Entity>List` envelope and the status code records of such an entity hold the wire type.
The generated functions `transform<Entity>ToPayload`, `transformPayloadTo<Entity>Insert` and
`transformPayloadTo<Entity>Update` convert between the persist types and the wire types with
functions from `ballerina/dataservice`, so a fix to a format is released without regenerating
the service:

| Type | To JSON | From JSON |
|---|---|---|
| `byte[]` | `dataservice:bytesToString` | `dataservice:bytesFromString` |
| `time:Date` | `dataservice:dateToString` | `dataservice:dateFromString` |
| `time:TimeOfDay` | `dataservice:timeOfDayToString` | `dataservice:timeOfDayFromString` |
| `time:Civil` | `dataservice:civilToString` | `dataservice:civilFromString` |
| `time:Utc` | `dataservice:utcToString` | `dataservice:utcFromString` |

A request value that does not parse is answered with `400` and `INVALID_INPUT`; see
[Errors](#errors).

#### Field rules

The field rules of #1503 shape the wire types:

| Rule | `<Entity>Payload` | `<Entity>InsertPayload` | `<Entity>UpdatePayload` |
|---|---|---|---|
| `ReadOnly` | Holds the field | Holds the field as optional, and the service ignores its value | Holds the field as optional, and the service ignores its value |
| `Immutable` | Holds the field | Holds the field | Does not hold the field |
| `Internal` | Does not hold the field | Does not hold the field | Does not hold the field |

- **Rejected fields:** the wire types are closed records, so a request that sets a field its
  wire type does not hold, an `Internal` field or an `Immutable` field on patch, is answered by
  `ballerina/http` with `400` and its default body.
- **Values on create:** `transformPayloadTo<Entity>Insert` sets a `ReadOnly` or `Internal`
  field to the `value` of its annotation, or to `()`.
- **Other types:** an `Internal` field is also left out of `<Entity>Filter`, `<Entity>Partial`
  and `<Entity>RelatedPartial`, so it cannot be filtered, sorted or selected.

### Query parameters

The query options of #1503 are query parameters of get many and of get related on a to-many
relation; get one and get related on a to-one relation take only `fields`:

| Parameter | Type | Example |
|---|---|---|
| `filter` | a JSON object, bound to the generated `<Entity>Filter` record | `filter={"hiredOn":{"gte":"2024-01-01"},"status":{"anyOf":["ACTIVE"]}}` |
| `sort` | field names, repeated or comma-separated; `-` before a name sorts it descending | `sort=name&sort=-hiredOn` or `sort=name,-hiredOn` |
| `size` | an integer | `size=20` |
| `offset` | an integer, the number of records to skip | `offset=40` |
| `fields` | field names, repeated or comma-separated | `fields=name,email` |

Repeated and comma-separated forms bind to the same `string[]`. ✅ Values in the URL are
percent-encoded as usual.

#### Filter

`<Entity>Filter` has one optional field per filterable entity field. Each is a record of the
operators that the field's type allows, and holds values of the field's wire type:

```ballerina
public type EmployeeFilter record {|
    record {| int eq?; int ne?; int lt?; int lte?; int gt?; int gte?; int[] anyOf?; |} id?;
    record {| string eq?; string ne?; string lt?; string lte?; string gt?; string gte?;
        string[] anyOf?; string contains?; string startsWith?; |} name?;
    record {| string eq?; string ne?; string lt?; string lte?; string gt?; string gte?;
        string[] anyOf?; string contains?; string startsWith?; boolean isNull?; |} email?;
    record {| string eq?; string ne?; string lt?; string lte?; string gt?; string gte?;
        string[] anyOf?; |} hiredOn?;
    record {| Status eq?; Status ne?; Status[] anyOf?; |} status?;
    record {| int eq?; int ne?; int lt?; int lte?; int gt?; int gte?; int[] anyOf?; |}
        departmentId?;
|};
```

- **Several operators on one field** all apply, so `{"id":{"gte":2,"lt":10}}` is a range.
- **Strict binding:** the records are closed, so `ballerina/http` answers an unknown field, an
  operator the type does not allow, a value of the wrong type or an unknown enum value with
  `400` before the resource runs. ✅
- **Dates and times** are strings in the filter, as in the body, and are parsed by
  `ballerina/dataservice`; a value that does not parse is answered with `INVALID_INPUT`.

#### Sort, page and fields

`sort` and `fields` are validated by `ballerina/dataservice` against the entity description, so
an unknown or unusable field is answered with `INVALID_INPUT` and the message #1503 gives.

```
GET /api/employees?sort=-hiredOn&size=2
→ 200 {"items": [{"id": 7, ...}, {"id": 3, ...}], "hasMore": true}

GET /api/employees?sort=-hiredOn&size=2&offset=2
→ 200 {"items": [{"id": 5, ...}], "hasMore": false}

GET /api/employees/7?fields=name
→ 200 {"id": 7, "name": "Ann"}
```

With `fields`, a record is returned as `<Entity>Partial` and a page as `<Entity>PartialList`.
`<Entity>Partial` has the fields of the entity's wire type and one field per relation. The key
fields are required and the others optional. A to-one relation is a nullable
`<Target>RelatedPartial`, and a to-many relation a `<Target>RelatedPartialList`:

```ballerina
public type EmployeePartial record {|
    int id;
    string name?;
    string? email?;
    string hiredOn?;
    Status status?;
    int departmentId?;
    DepartmentRelatedPartial? department?;
|};

public type DepartmentPartial record {|
    int id;
    string name?;
    EmployeeRelatedPartialList employees?;
|};

public type DepartmentRelatedPartial record {|
    int id;
    string name?;
|};

public type EmployeeRelatedPartialList record {|
    EmployeeRelatedPartial[] items;
    boolean hasMore;
|};
```

`fields=` with no value is answered with `400` and `INVALID_INPUT`.

### Relations

Each relation of an exposed entity gets a resource under the entity's key, and can be named in
`fields`, when both the entity and the related entity allow `read`, as #1503 defines:

```
GET /api/employees/4/department
→ 200 {"id": 2, "name": "Research"}

GET /api/departments/2/employees?sort=name&size=2
→ 200 {"items": [{"id": 4, ...}, {"id": 9, ...}], "hasMore": true}

GET /api/employees?fields=name,department.name
→ 200 {"items": [{"id": 4, "name": "Ann", "department": {"id": 2, "name": "Research"}}, ...], "hasMore": false}

GET /api/departments/2?fields=name,employees.name
→ 200 {"id": 2, "name": "Research", "employees": {"items": [{"id": 4, "name": "Ann"}, ...], "hasMore": true}}
```

- **Segment:** the relation field's name in the model, unchanged, follows the entity's full key:
  `/assignments/4/P1/employee` for a composite key. Every entity has a fixed number of key
  segments, so a relation resource never matches the same path as get one.
- **Dotted fields:** in `fields`, a relation's own fields are written `<relation>.<field>`, as
  Directus does, on to-one and to-many relations. A relation named alone includes the whole
  related record.
- **To-many in `fields`:** an included to-many relation is a `{"items": [...], "hasMore": ...}`
  object. When `hasMore` is `true`, the caller pages through the rest with get related.
- **Empty to-one in `fields`:** a to-one relation with no related record is included as
  `"department": null`.
- **Not found:** a request is answered with `404` and `NOT_FOUND` when the record does not
  exist, or when a to-one relation has no related record, with the messages #1503 gives.

```ballerina
resource function get employees/[int id]/department(string[]? fields = ())
        returns Department|DepartmentPartial|dsHttp:NotFound|dsHttp:InvalidInput|dsHttp:Forbidden
            |dsHttp:InternalError { ... }

resource function get departments/[int id]/employees(EmployeeFilter? filter = (),
        string[]? sort = (), int? size = (), int? offset = (), string[]? fields = ())
        returns EmployeeList|EmployeePartialList|dsHttp:NotFound|dsHttp:InvalidInput
            |dsHttp:Forbidden|dsHttp:InternalError { ... }
```

### Errors

Each error category in #1503 maps to a status code. The body is the `DataserviceError` record
from `ballerina/dataservice`, with the code and message #1503 defines for the category.

| Category | Status | `code` | Raised from |
|---|---|---|---|
| Not found | `404` | `NOT_FOUND` | `persist:NotFoundError` ✅ |
| Conflict | `409` | `CONFLICT` | `persist:AlreadyExistsError` ✅ |
| Constraint violation | `409` | `CONSTRAINT_VIOLATION` | `persist:ConstraintViolationError` |
| Invalid input | `400` | `INVALID_INPUT` | a date, time or `byte[]` value that does not parse; a constraint broken on a field of a wire type; an unusable `sort` or `fields` name, a field named twice in `sort`, an empty `fields`, `size` below 1, `offset` below 0 or too large for the `size`; a create above `dataserviceMaxPageSize` records; a `float` key that is not finite |
| Invalid input | `400` | — | `ballerina/http` payload or path binding ✅; a constraint broken on an entity without wire types |
| Internal | `500` | `INTERNAL` | any other error |
| Unauthenticated | `401` | — | `ballerina/http` auth, before the resource runs ✅ |
| Missing scope | `403` | — | `ballerina/http` auth, before the resource runs ✅ |
| Forbidden | `403` | `FORBIDDEN` | get related or a relation in `fields` whose `read` scopes the entity's `read` scopes do not cover, when `auth` is `true`; declared only on the resources that can return it, see [Generated resources](#generated-resources) |

The status code records are not specific to an entity, so they are defined in
`ballerina/dataservice.http`, which keeps `ballerina/dataservice` free of `ballerina/http`:

```ballerina
public type NotFound record {|
    *http:NotFound;
    dataservice:DataserviceError body;
|};
```

`InvalidInput`, `Conflict`, `ConstraintViolation`, `Forbidden` and `InternalError` follow the
same form.

An error whose category is not among the error records of its resource is answered with `500`
and `INTERNAL`, and logged.

The message for a value that does not parse is
`The field '<field>' of the entity '<Entity>' is not <format>.`, where `<format>` is
`an ISO 8601 date`, `an ISO 8601 time`, `an ISO 8601 date-time` or `a base64 string`.

Other invalid input is answered by `ballerina/http` before the resource runs, so its body is
the module's default error body, not `DataserviceError`. For a request body, it names every
missing or mistyped field. ✅ For `filter`, it does not; see
[Risks and assumptions](#risks-and-assumptions).

Example responses: ✅

```
POST /api/employees  {"id":1, ...}            → 201  Location: /api/employees/1
POST /api/employees  {"id":1, ...} (again)    → 409  {"code":"CONFLICT", "message":"A record that conflicts with an existing record of the entity 'Employee' already exists."}
GET  /api/employees/99                        → 404  {"code":"NOT_FOUND", "message":"A record with the key '99' does not exist for the entity 'Employee'."}
GET  /api/employees/abc                       → 400  (ballerina/http default body)
PUT  /api/employees/2                         → 405
```

### Auth

With `options.dataservice.auth = false`, the default, the service has no auth annotations and
`dataservice_config.bal` declares no `dataserviceAuth`.

With `auth = true`, the service passes `dataserviceAuthConfig`, the checked value of
`dataserviceAuth` from [Configuration](#configuration), to `@http:ServiceConfig`, and each
resource requires the scopes of its action through `@http:ResourceConfig`, which accepts scopes
alone and takes the providers from the service:

```ballerina
@http:ServiceConfig {auth: dataserviceAuthConfig}
service /api on dataserviceListener {

    @http:ResourceConfig {auth: {scopes: ["hr:read"]}}
    resource function get employees(...) returns ... { ... }
}
```

- **Required providers:** `dataserviceAuth` is a required configurable value without a
  default, so the program does not start when `Config.toml` does not set it. An empty list
  also stops it at startup, through a module-level `check` on `dsHttp:authConfig`, with the
  message
  `The configurable value 'dataserviceAuth' must hold at least one authentication provider.`
- **Authentication only:** an action with an empty scope list gets no `@http:ResourceConfig`
  auth, so the service's providers apply alone.
- **Responses:** `ballerina/http` answers a missing or invalid credential with `401` and a
  missing scope with `403`, with its default body, before the resource runs. ✅ On Ballerina
  2201.14.0, with `http:ListenerAuthConfig[]` read from `Config.toml` as basic auth with a file
  user store, a request without credentials was answered with `401`, a user without the scope
  with `403`, and a user with the scope with `200`; the `401` body was `ballerina/http`'s
  default error body.
- **Get related** and a relation in `fields` require the entity's `read` scopes, and are
  answered with `403` and `FORBIDDEN` when those do not cover the related entity's, as #1503
  defines.

For example, to validate JWTs, set in `Config.toml`:

```toml
[[dataserviceAuth]]
[dataserviceAuth.jwtValidatorConfig]
issuer = "https://idp.example.com"
audience = "hr-api"
[dataserviceAuth.jwtValidatorConfig.signatureConfig.jwksConfig]
url = "https://idp.example.com/jwks"
```

### Constraints

The constraints of the model, kept in the generated types, are checked by `ballerina/http`'s
payload validation, which answers a value that breaks one with `400` and its default body.
Wire types copy the model's constraint annotations. A wire type holds dates, times and `byte[]`
values as strings, so the service checks the constraints on those fields on the converted
record, and answers a failure with `400`, `INVALID_INPUT` and the message #1503 gives,
`The field '<field>' of the entity '<Entity>' breaks its '<constraint>' constraint.`

### OpenAPI definition

Unless `options.dataservice.introspection` is `false`, the service carries
`@openapi:ServiceInfo {embed: true}`. The compiler then embeds an OpenAPI definition built from
the service and its doc comments, and `ballerina/http` serves it with Swagger UI: ✅

| Request | Response |
|---|---|
| `OPTIONS /api` | A `Link` header, `</api/openapi-doc-dygixywsw>;rel="service-desc", </api/swagger-ui-dygixywsw>;rel="swagger-ui"` |
| `GET /api/openapi-doc-dygixywsw` | The OpenAPI definition. Operation summaries come from the resources' doc comments, and schema and property descriptions from the types' doc comments |
| `GET /api/swagger-ui-dygixywsw` | Swagger UI for the definition |

- **Paths:** the two paths are fixed by `ballerina/http` and cannot be changed. A client finds
  the definition through the `Link` header, whose `service-desc` relation is the standard one
  for a service description (RFC 8631).
- **Swagger UI** loads its scripts and styles from `unpkg.com`, so it needs internet access in
  the browser. The definition itself does not.
- **Auth:** `ballerina/http` serves the definition, Swagger UI and the `OPTIONS` `Link` header
  without applying the service's auth. ✅ On Ballerina 2201.14.0, with basic auth configured
  from `Config.toml`, all three were answered with `200` without credentials. `introspection`
  therefore defaults to `false` when `options.dataservice.auth` is `true`; an explicit value
  applies in both cases.

### Configuration

| Setting | Where | Default |
|---|---|---|
| Base path | `options.dataservice.basePath` | `/api` |
| Auth | `options.dataservice.auth` | `false` |
| Publishing the OpenAPI definition | `options.dataservice.introspection` | `true`, or `false` when `auth` is `true` |
| Port | `dataservicePort` in `Config.toml` | `9090` |
| Page size when `size` is not given | `dataservicePageSize` in `Config.toml` | `100` |
| Largest page size | `dataserviceMaxPageSize` in `Config.toml` | `1000` |
| Authentication providers | `dataserviceAuth` in `Config.toml` | none: required when `auth` is `true`, and not declared otherwise |
| Other listener settings, such as TLS and timeouts | `dataserviceListenerConfig` in `Config.toml` | `ballerina/http`'s defaults |

`dataservice_config.bal` declares the configurable values:

```ballerina
configurable int dataservicePort = 9090;
configurable http:ListenerConfiguration dataserviceListenerConfig = {};
configurable int dataservicePageSize = 100;
configurable int dataserviceMaxPageSize = 1000;
// Only with options.dataservice.auth = true
configurable http:ListenerAuthConfig[] dataserviceAuth = ?;

final dataservice:PageConfig dataservicePageConfig =
        check dataservice:pageConfig(dataservicePageSize, dataserviceMaxPageSize);
// Only with options.dataservice.auth = true
final http:ListenerAuthConfig[] dataserviceAuthConfig = check dsHttp:authConfig(dataserviceAuth);
```

The program stops at startup, with the messages #1503 gives, when a page size is out of range.

For example, to serve over TLS:

```toml
[dataserviceListenerConfig.secureSocket.key]
certFile = "/path/to/public.crt"
keyFile = "/path/to/private.key"
```

### Generated code

#### Generated resources

```ballerina
service /api on dataserviceListener {

    resource function get employees(EmployeeFilter? filter = (), string[]? sort = (), int? size = (),
            int? offset = (), string[]? fields = ())
            returns EmployeeList|EmployeePartialList|dsHttp:InvalidInput|dsHttp:Forbidden
                |dsHttp:InternalError {
        // Validates the options and builds the query with ballerina/dataservice, reads one page
        // through the persist client, and returns it with hasMore.
    }

    resource function get employees/[int id](string[]? fields = ())
            returns Employee|EmployeePartial|dsHttp:NotFound|dsHttp:InvalidInput|dsHttp:Forbidden
                |dsHttp:InternalError { ... }

    resource function post employees(EmployeeInsert|EmployeeInsert[] payload)
            returns EmployeeCreated|EmployeeListCreated|dsHttp:InvalidInput|dsHttp:Conflict
                |dsHttp:ConstraintViolation|dsHttp:InternalError {
        // In one transaction, posts the records as an array and reads back the created records
        // with one query on their keys.
    }

    resource function patch employees/[int id](EmployeeUpdate payload)
            returns Employee|dsHttp:NotFound|dsHttp:InvalidInput|dsHttp:Conflict
                |dsHttp:ConstraintViolation|dsHttp:InternalError { ... }

    resource function delete employees/[int id]()
            returns Employee|dsHttp:NotFound|dsHttp:InvalidInput|dsHttp:ConstraintViolation
                |dsHttp:InternalError { ... }
}
```

- **Wire types:** the signatures show the persist types. An entity with a date, time or
  `byte[]` field, such as `Employee` with `hiredOn`, uses its wire types in their place, such
  as `EmployeePayload`; see [Wire types](#wire-types).
- **Without `read`:** patch and delete return `http:NoContent` in place of the record.
- **Return types:** each resource lists the error records it can return, so an OpenAPI
  definition generated from the service shows exactly its status codes:

  | Operation | Error records |
  |---|---|
  | Get many | `InvalidInput`, `Forbidden` when it can refuse a relation, `InternalError` |
  | Get one, get related | `NotFound`, `InvalidInput`, `Forbidden` when it can refuse a relation, `InternalError` |
  | Create | `InvalidInput`, `Conflict`, `ConstraintViolation`, `InternalError` |
  | Patch | `NotFound`, `InvalidInput`, `Conflict`, `ConstraintViolation`, `InternalError` |
  | Delete | `NotFound`, `InvalidInput`, `ConstraintViolation`, `InternalError` |

  The `401` and `403` responses of `ballerina/http` auth are not among them; see
  [Risks and assumptions](#risks-and-assumptions).
- **`Forbidden`:** declared only on a resource that can refuse a relation: with `auth = true`,
  get many and get one of an entity with a relation whose `read` scopes its own do not cover,
  and get related on such a relation. Other resources never return it, so their OpenAPI
  definition lists no such `403`. The signatures above declare it, as in a service with
  `auth = true` where the `read` scopes of `Employee` do not cover those of `Department`.
- **Library calls:** the generated service imports `ballerina/dataservice.http` as `dsHttp`
  and passes the entity name and key values to its functions, which build the error response,
  log the error detail and build the `Location` header.
- **Location:** each key value is percent-encoded as a path segment, so a key such as `P 1/a`
  gives `/api/assignments/4/P%201%2Fa`. ✅ With the base path `/`, `Location` starts at the
  entity, as in `/employees/1`.

The status code records of an entity include the `ballerina/http` status code records:

```ballerina
public type EmployeeCreated record {|
    *http:Created;
    Employee body;
|};

public type EmployeeListCreated record {|
    *http:Created;
    EmployeeList body;
|};
```

#### Generated source

- **Header:** each generated file starts with the header of persist's generated files, stating
  that the file is generated and must not be edited. A service written by
  `bal persist generate` has the same header without the instruction not to edit it.
- **Order:** `dataservice.bal` holds the client and the listener, then one service with the
  resources of each exposed entity in model order. Within an entity, the order is get many, get
  one, get related (in model order of the relation fields), create, patch and delete.
  `dataservice_types.bal` holds, per entity in model order, the wire types, the `<Entity>List`
  envelope, the filter and partial types, the status code records and then the functions that
  convert to and from the wire types.
- **Doc comments:** every resource and every generated public type has a short doc comment: a
  one-line summary and one-line `+ param`, `+ return` and `+ field` lines. The summary of a
  resource comes from its operation, and the entity's doc comment from the model follows it:

  | Operation | Summary |
  |---|---|
  | Get many | `Gets a page of employees.` |
  | Get one | `Gets an employee by its key.` |
  | Get related | `Gets the department of an employee.` |
  | Create | `Creates one or more employees.` |
  | Patch | `Updates an employee by its key.` |
  | Delete | `Deletes an employee by its key.` |

  A field's `+ field` line is its doc comment in the model.

## Alternatives

- **A separate route for create many**, such as `POST /employees/$batch`. Not needed: the
  union payload type lets one route take both, and `$` names stay reserved for query and
  metadata operations.
- **`PUT` for update.** `PUT` means full replace in HTTP, and the persist client's update is
  partial, so the method is `PATCH`.
- **`204 No Content` for every delete.** When the entity allows `read`, returning the deleted
  record matches the persist client and the other protocols, and costs nothing.
- **A bare array for get many.** Rejected in #1503: paging and count need room in the
  envelope.
- **Other composite key forms.** Named pairs, as Data API builder uses
  (`/assignments/employeeId/4/projectCode/P1`), say which value is which key but make single
  keys unusual (`/employees/id/4`). OData's `/assignments(employeeId=4,projectCode='P1')` has no
  form in a Ballerina resource path. A delimiter in one segment (`/assignments/4,P1`) needs
  escaping and cannot be typed. Query parameters (`/assignments?employeeId=4&projectCode=P1`)
  address the collection, where filters belong.

## Testing

- Code generation: expected-output tests for single and composite keys, every action
  combination, and a custom base path.
- Behaviour, against H2 and every other SQL datastore available to the test run: every row of
  the operations and errors tables, including `405` for a disabled method and `400` for a
  mistyped key; `204` for patch, delete and `PATCH {}` on an entity without `read`; `201` with
  `{"items": [], "hasMore": false}` for `POST []`; a create many with a duplicate key among its
  records, which creates none; a create above `dataserviceMaxPageSize` records.
- Keys: a `Location` round trip for a `float` key, and `400` for a `NaN` or `Infinity` key.
- Startup: an out-of-range page size, and with `auth = true`, a missing or empty
  `dataserviceAuth`, each stopping the program with its message.
- The OpenAPI definition: the `Link` header, the definition with the generated summaries and
  the model's descriptions, Swagger UI, and none of them with `introspection = false`.
- Field rules: per rule, the wire types generated, a `ReadOnly` field ignored on create and
  patch, an `Immutable` field rejected on patch, an `Internal` field rejected and never returned,
  and `value` set on create.
- A round trip of every `time:*` type and `byte[]` through create and get one, and `400` with
  `INVALID_INPUT` for a value in the wrong format.
- Query parameters, against H2 and every other SQL datastore available to the test run: every
  operator on every type it allows, `ne` on a nullable field, an empty `anyOf`, `contains` and
  `startsWith` with `%`, `_` and `[`, multi-field sort with nulls, a field named twice in
  `sort`, paging through every page by `offset` until `hasMore` is `false`, `size` above the
  maximum, `offset` below 0 and too large for the `size`, and `fields`, including a repeated
  field, an empty `fields=`, and a relation named with one of its fields.
- Relations, against H2 and every other SQL datastore available to the test run: get related on
  to-one and to-many relations, single and composite keys, a missing record and an empty to-one
  relation; one and several relations in `fields`, to-one and to-many, with dotted fields and a
  to-many list longer than the page size; an empty to-one relation and one pointing at a
  missing record, as `null`; a relation of a related entity in `fields`, rejected; the number
  of queries per request, as #1503 gives it.
- Auth: `auth = false`, with no auth code generated; a credential without a scope (`403`), with
  a default scope, with a scope from `scopes`, and with an empty scope list; no credential
  (`401`); get related and a relation in `fields` allowed when the entity's `read` scopes cover
  the related entity's, and answered with `403` and `FORBIDDEN` when they do not; the OpenAPI
  definition not published by default with `auth = true`, and published when `introspection`
  is set to `true`.
- Constraints: a create and a patch that break a constraint, on an entity with and without wire
  types.

## Risks and assumptions

- **Two error body shapes for `400`.** Malformed or mistyped input, and a constraint broken on
  an entity without wire types, are rejected by `ballerina/http`'s payload validation with its
  default body. A constraint broken on a field of a wire type, and the other errors, use
  `DataserviceError`. For `filter`, the default body says only
  `error in casting query param : 'filter'`, without naming the field or operator. ✅ An error
  interceptor on the generated service, which is future work, aligns the two shapes.
- **`401` and `403` are not in the OpenAPI definition.** They come from `ballerina/http` auth
  and are not in the resources' return types, so the definition does not list them.
- **Lost updates.** As in #1503, there is no `ETag` or `If-Match`, so concurrent `PATCH`
  requests on one record can overwrite each other.

## Dependencies

- [Data service generation for bal persist](https://github.com/ballerina-platform/ballerina-spec/issues/1503)
- `ballerina/http`
- `ballerina/openapi`, for `@openapi:ServiceInfo`
- `ballerina/dataservice`
- `ballerinax/persist.sql` classifying a foreign key violation on insert and delete as
  `persist:ConstraintViolationError`
  ([ballerina-library#9359](https://github.com/ballerina-platform/ballerina-library/issues/9359)),
  for `409` with `CONSTRAINT_VIOLATION` on create and delete
- `ballerina/dataservice.http`: a new library, released separately from `ballerina/dataservice`

## Future work

- An error interceptor so invalid-input errors use `DataserviceError`.

