# Data service generation for bal persist

- Author: @niveathika
- Reviewers: 
- Created: 2026-09-18
- Updated: 2026-10-07
- Issue: [#1503](https://github.com/ballerina-platform/ballerina-spec/issues/1503)
- Status: Submitted

## Summary

This proposal lets `bal persist` generate a data service over an existing persist model. The
user adds one options group to the existing `[[tool.persist]]` entry, naming the protocol and,
optionally, the entities and operations to expose. The tool then generates a listener and a
service alongside the persist client.

```
bal persist add --datastore postgresql --dataservice http --entities Employee,Department
# write persist/model.bal, or bal persist pull
bal build                     # client and data service are generated
```

The service has a fixed CRUD shape: six operations per entity, grouped into four actions. This
proposal defines that shape, how the tool is configured and how it behaves. It does not define
how each protocol renders the shape. That is left to follow-up proposals for HTTP, GraphQL and
MCP, listed under [Future work](#future-work).

This is proposal 1 of 4.

Please add any comments to issue [#1503](https://github.com/ballerina-platform/ballerina-spec/issues/1503).

## Goals

- Generate a working data service from an existing persist model, with no hand-written service
  code.
- Define one protocol-neutral CRUD shape that every protocol renders.
- Configure the service through the existing `[[tool.persist]]` entry and existing commands. No
  new `bal persist` command.
- Let the model restrict what each entity allows, using annotations that are checked at
  compile time.
- Leave persist's behaviour unchanged when no data service is configured.
- Fix the parts of the contract that later features (query options, auth, relations) would
  otherwise break: the response envelope, the action names, the addressing scheme, the write
  results and the error categories.

## Non-goals

- Query options: filtering, sorting, paging, field selection and count.
- Relations: embedded reads, relation routes and nested writes.
- Authentication and authorization.
- Write operations the persist client does not provide: upsert, full replace, and update or
  delete by filter.
- Exposing several protocols from one entry.
- A data service in a module other than the package's default module, or over more than one
  persist model.
- OData and SOAP.

All of these are in the roadmap under [Future work](#future-work). The design below reserves
room for each so that adding them does not break v1 services.

## Motivation

A persist model already describes the entities, their keys and their types, and the generated
persist client already implements create, read, update and delete for each one. What is
missing is the service layer: today a user who wants an API over their data writes an HTTP,
GraphQL or MCP service by hand, resource by resource, wiring each one to the client and mapping
its errors.

Comparable tools generate this layer:

| Tool | Source of truth | Protocols generated |
|---|---|---|
| [Azure Data API builder](https://learn.microsoft.com/azure/data-api-builder/) | `dab-config.json` | REST, GraphQL and MCP, from one process |
| [ZenStack](https://zenstack.dev/) | ZModel schema | REST (RPC style and JSON:API); MCP through a community package |
| [PostgREST](https://docs.postgrest.org/) | The database | REST; GraphQL and MCP only through separate products |

All three expose the same per-entity operation set: list, get by key, create, update and
delete. All three also control access in the same four actions: create, read, update and
delete. That is the shape adopted here.

For a single entity, the operations each tool offers compare with the persist client as
follows:

| Operation | ZenStack | Data API builder | PostgREST | persist client |
|---|---|---|---|---|
| List | yes | yes | yes | yes |
| Get by key | yes | yes | yes | yes |
| Create, including bulk | yes | yes | yes | yes, bulk only |
| Partial update by key | yes | yes | yes | yes |
| Delete by key | yes | yes | yes | yes |
| Upsert | yes | yes | yes | no |
| Update or delete by filter | yes | no | yes | no |

Everything the persist client supports is in scope. The last two rows are future work, and
depend on the persist client first.

## Design

### Overview

| Piece | Owner | Contents |
|---|---|---|
| `Ballerina.toml` | user | Whether a data service exists, its protocol, base path, entities and default operations |
| `persist/model.bal` | user | The entities, plus optional `@dataservice:Expose` annotations restricting an entity |
| `ballerina/dataservice` | new library | The annotations, their compiler-plugin validation, and the runtime types generated services share |
| `generated/` | persist tool | The persist client, as today, plus the data service |

### `Ballerina.toml`

The data service is an options group on the existing `[[tool.persist]]` entry.

```toml
[[tool.persist]]
id = "generate-db-client"
targetModule = "hr"
filePath = "persist/model.bal"
options.datastore = "postgresql"
options.dataservice.protocol = "http"
options.dataservice.basePath = "/api"
options.dataservice.entities = ["Employee", "Department"]
options.dataservice.operations = ["read", "create", "update", "delete"]
```

| Key | Required | Default | Meaning |
|---|---|---|---|
| `options.dataservice.protocol` | yes | — | The protocol the service is generated for: `http`, `graphql` or `mcp` |
| `options.dataservice.basePath` | no | `/api` for `http`, `/graphql` for `graphql`, `/mcp` for `mcp` | The service's base path |
| `options.dataservice.entities` | no | All entities in the model | The entities to expose, by entity name |
| `options.dataservice.operations` | no | `["read", "create", "update", "delete"]` | The actions every exposed entity allows, unless its annotation narrows them |

When `options.dataservice` is absent, no service is generated and persist behaves exactly as it
does today. When it is present, `protocol` must be set.

#### Protocol values

Each value names the Ballerina module that implements the service:

| Value | Generated service | Listener |
|---|---|---|
| `http` | `http:Service` | `http:Listener` |
| `graphql` | `graphql:Service` | `graphql:Listener` |
| `mcp` | `mcp:StreamableHttpService` | `mcp:StreamableHttpListener` |

`http` is used rather than `rest` because it names the protocol and the module, where `rest`
names a design style. GraphQL and MCP are also carried over HTTP; the value selects the service
type, not the transport.

`odata` and `soap` are reserved for later proposals. Passing them now is an error that lists
the supported values.

`protocol` takes a single string. A later proposal may also accept an array, generating one
service per protocol on a shared listener. A string stays valid when that happens.

#### Entities

`entities` is an include list. When omitted, every entity in the model is exposed, and the tool
reports which entities it exposed on each generation so that an entity added to the model is
not exposed unnoticed.

An empty list is an error. A name that is not an entity in the model is an error.

### CLI

No new command is added. Two flags are added to `add` and `pull`.

| Flag | Writes | Meaning |
|---|---|---|
| `--dataservice <protocol>` | `options.dataservice.protocol` | Enables the data service for the given protocol |
| `--entities <Entity,...>` | `options.dataservice.entities` | The entities to expose. Requires `--dataservice` |

`basePath` and `operations` have no flags. They have defaults and are edited in
`Ballerina.toml`.

| Command | Behaviour with `--dataservice` |
|---|---|
| `bal persist add` | Writes the `[[tool.persist]]` entry as today, and adds the `options.dataservice` keys to it. `--entities` is not checked against the model here, because `add` writes an empty model |
| `bal persist pull` | Introspects the database and writes the model as today. Then records the `options.dataservice` keys in the existing `[[tool.persist]]` entry, creating the entry as `add` would if none exists. Without `--entities`, every pulled entity is exposed; `--tables` already selects which tables are pulled. With `--entities`, the names are checked against the pulled model |
| `bal build` | Generates the client and, when `options.dataservice` is present, the data service |

`init`, `generate`, `migrate` and `push` are unchanged. `init` and `generate` are the one-time
generation path that predates `add`: they do not read `[[tool.persist]]` and do not generate a
data service. The data service is generated only through the `[[tool.persist]]` entry, at build
time.

`pull --entities` names model entities, not database tables. `pull --tables` keeps its current
meaning, so a user can pull more tables than they expose.

### The `ballerina/dataservice` module

The module holds:

- the annotations a model uses to restrict an entity,
- a compiler plugin that validates them, and
- the protocol-neutral runtime types that generated services share: the error body and its
  error codes.

Keeping the runtime types in a library rather than in generated code means the generated code
stays thin, and fixes ship as library updates without regeneration. The library does not depend
on any protocol module. Types that are specific to an entity, such as the `<Entity>List`
envelope, or to a protocol, such as an HTTP status code record, are generated.

A model that uses the annotations imports the module with a prefix:

```ballerina
import ballerina/persist as _;
import ballerina/dataservice;

@dataservice:Expose {operations: [dataservice:READ]}
type Department record {|
    readonly int id;
    string name;
|};

type Employee record {|
    readonly int id;
    string name;
    string email;
|};
```

The import is needed only when the model uses an annotation. It is not a switch: whether a
service is generated is decided by `options.dataservice` in `Ballerina.toml`. The persist
tool's model validation accepts this import alongside the ones it accepts today.

#### `@dataservice:Expose`

```ballerina
public enum Operation {
    READ = "read",
    CREATE = "create",
    UPDATE = "update",
    DELETE = "delete"
}

public type ExposeConfig record {|
    Operation[] operations;
|};

public annotation ExposeConfig Expose on type;
```

The annotation narrows what an entity allows. An entity's effective operations are those listed
in both `options.dataservice.operations` and its annotation. An annotation never widens what
`Ballerina.toml` grants.

| `Ballerina.toml` operations | Annotation | Effective |
|---|---|---|
| `read, create, update, delete` | none | `read, create, update, delete` |
| `read, create, update, delete` | `[READ]` | `read` |
| `read` | `[READ, UPDATE]` | `read` |

An entity whose effective operations are empty is an error.

The member values match the strings `Ballerina.toml` uses. `EXECUTE` is reserved for custom
operations in a later proposal.

#### Validation

The compiler plugin checks the annotation where it is written. The persist tool checks it
against `Ballerina.toml` when it generates.

| Check | Where | Severity |
|---|---|---|
| `@dataservice:Expose` is on a record type that is a persist entity | compiler plugin | error |
| `operations` is non-empty and has no duplicates | compiler plugin | error |
| The annotated entity is not in `options.dataservice.entities` | persist tool | warning |
| An entity's effective operations are empty | persist tool | error |

Diagnostic codes and messages are left to the implementation.

### The service shape

#### Actions and operations

Each exposed entity gets up to six operations, enabled in four groups called actions:

| Action | Operation | Persist client call |
|---|---|---|
| `read` | Get many | `get <entities>()` |
| `read` | Get one | `get <entities>/[key]()` |
| `create` | Create one | `post <entities>([value])`, then get one by the returned key |
| `create` | Create many | `post <entities>(values)`, then get one for each returned key |
| `update` | Patch one | `put <entities>/[key](value)` |
| `delete` | Delete one | `delete <entities>/[key]()` |

The two operations in an action expose the same data, so they are enabled together. Splitting
them would not restrict what a caller can see or change.

The persist client's `put` is a partial update: fields left out of the value are not changed.
It is exposed as a patch.

#### Addressing

- **Collection name:** the resource name the persist client already uses for the entity, for
  example `employees` for `Employee`.
- **Key:** the entity's `readonly` fields, in declaration order. Every persist entity has a key,
  so get one, patch one and delete one exist for every entity whose actions allow them.
- **Composite keys:** take one value per key field, in that order.

These are reserved for later proposals and are not used for anything else:

- a third path segment after an entity's key (`<entities>/[key]/<relation>`), for relations;
- names starting with `$`, for query and metadata operations such as count.

#### Types

The shape is defined in terms of the types persist already generates. The HTTP and MCP services
use them directly; the GraphQL service generates equivalent input and output types, because
`ballerina/graphql` does not accept the persist types as they are (see the GraphQL proposal).

| Type | Used as |
|---|---|
| `<Entity>` | The result of every operation that returns a record |
| `<Entity>Insert` | The input to create one and create many |
| `<Entity>Update` | The input to patch one |

Get many and create many return an envelope rather than a bare array:

```ballerina
public type EmployeeList record {|
    Employee[] items;
|};
```

The envelope exists so that paging and count can add fields such as a next-page cursor and a
total later, without changing the result type. MCP also requires a tool's structured result to
be an object, so a bare array could not be used there in any case.

#### Results

| Operation | Returns |
|---|---|
| Get many | `<Entity>List` holding every record |
| Get one | `<Entity>` |
| Create one | The created `<Entity>`, read back after insert so that generated fields are included |
| Create many | `<Entity>List` holding the created records |
| Patch one | The updated `<Entity>`, as returned by the persist client |
| Delete one | The deleted `<Entity>`, as returned by the persist client |

Creates return records rather than keys because changing a result type later would break
clients.

#### Errors

Errors fall into protocol-neutral categories. `ballerina/dataservice` defines one error body
and an error code per category:

```ballerina
public enum ErrorCode {
    NOT_FOUND,
    CONFLICT,
    CONSTRAINT_VIOLATION,
    INVALID_INPUT,
    INTERNAL
}

public type DataserviceError record {|
    ErrorCode code;
    string message;
|};
```

Each protocol proposal maps the categories to its own form: an HTTP status code, an entry in
the GraphQL `errors` array, or an MCP tool result with `isError` set.

| Category | `code` | Raised when | Source |
|---|---|---|---|
| Not found | `NOT_FOUND` | No record has the given key | `persist:NotFoundError` |
| Conflict | `CONFLICT` | A record with the key or a unique value already exists | `persist:AlreadyExistsError` |
| Constraint violation | `CONSTRAINT_VIOLATION` | A write violates a foreign key | `persist:ConstraintViolationError` |
| Invalid input | `INVALID_INPUT` | The input cannot be bound to the operation's type | The protocol module's binding error |
| Internal | `INTERNAL` | Any other failure | `persist:Error` and others |

The message is written by the generated service, not taken from the persist error, because the
SQL datastores put driver text, such as table and constraint names, in their error messages:

| Category | Message |
|---|---|
| Not found | `A record with the key '<key>' does not exist for the entity '<Entity>'.` |
| Conflict | `A record that conflicts with an existing record of the entity '<Entity>' already exists.` |
| Constraint violation | `The operation violates a constraint of the entity '<Entity>'.` |
| Internal | `An internal error occurred.` |

A composite key is written as its values joined by `/`, in key order. The persist error's own
detail for a constraint violation or an internal error is logged and not returned to the
caller.

A disabled operation is not generated at all, so it does not exist rather than being refused.
Which protocol error a caller sees when calling one is protocol-specific: a 404 or 405 for HTTP,
a validation error for GraphQL, an unknown tool for MCP.

### Generated code

- **Location:** the data service is generated only into the package's default module. When
  `options.dataservice` is present, `targetModule` must be the package name, which is persist's
  default when `--module` is not passed. Any other `targetModule` is an error.
- **Why the default module:** a service in the default module is started without the user
  importing anything.
- **Files:** the service is generated into `generated/` next to the persist client, in one file
  per entry. Like the client, it is regenerated on every build and must not be edited.
- **Names:** generated module-level names share the default module's namespace with the user's
  own code, as the persist client's generated names already do. The service follows the same
  convention: type names are derived from entity names, such as `<Entity>List`, and other
  module-level names start with `dataservice`.
- **One service per package:** a package has one persist model and at most one data service.
  Persist's `--model` option for several models is not supported together with
  `options.dataservice`.

#### Runtime configuration

Runtime settings follow the database connection settings. Persist generates the database host,
port, user, password and database name as configurable values in `persist_db_config.bal`, and
the user sets them in `Config.toml`. The data service does the same: the tool generates
`persist_dataservice_config.bal` holding the listener port as a configurable value.

```ballerina
configurable int dataservicePort = 9090;
```

The port has a default, unlike the database settings, so the service runs without any
`Config.toml` entry. The name is not `port` because the database configuration already declares
`port` in the same module.

Other listener settings, such as TLS, are left to the protocol proposals.

### Datastore support

The six operations use only client calls that every persist datastore implements: MySQL, MSSQL,
PostgreSQL, H2, in-memory, Google Sheets and Redis. The CRUD shape is therefore available on all
of them.

Whether each field type can be represented in each protocol is defined by the protocol
proposals. For example, persist's `time:*` record types and `byte[]` need a GraphQL mapping.

### Backward compatibility

- Without `options.dataservice`, nothing changes: the client, its files and all commands
  behave as today.
- `persist-options-schema.json` currently sets `additionalProperties: false`. It gains a
  `dataservice` object; existing entries still validate.
- Models that do not import `ballerina/dataservice` are unaffected.

## Alternatives

### A hand-written service spec file

A separate file, `persist/service.bal`, would declare the API as a service object type, kept in
step with the model by a new `bal persist expose` command. It can express more than annotations:
per-resource query capability, renamed operations, custom SQL-backed operations.

It is not adopted for this proposal because it adds a second hand-maintained artifact and a new
command to deliver a CRUD shape the model already determines. It remains an option for later
features that cannot be expressed as annotations.

### Per-entity configuration in `Ballerina.toml`

For example, `options.dataservice.entities.Department = ["read"]`. This splits what the data
allows across two files, and cannot express field-level rules such as a hidden field.
`Ballerina.toml` is kept for how and where the service is served; the model holds what the data
allows.

### A `readOnly` flag, or one switch per operation

A `readOnly` flag cannot express a common case such as "create and update, but not delete".
Six switches separate operations that expose the same data, adding configuration without
adding control. The four actions match the access models of Data API builder, ZenStack's access
policies and PostgreSQL's privileges.

### `rest` as the protocol value

See [Protocol values](#protocol-values).

### Several protocols from one entry

The listeners of `ballerina/graphql` and `ballerina/mcp` both accept an `http:Listener`, so
several services can share one port. This is deferred so that each protocol's rendering can be
specified and settled on its own first. `protocol` is a string that can later also accept an
array.

## Testing

- **Options schema:** `options.dataservice` with valid and invalid values; required
  `protocol`; reserved protocol values; empty and unknown `entities`.
- **CLI:** `add` and `pull` with and without `--dataservice` and `--entities`;
  `pull` into a project with and without an existing `[[tool.persist]]` entry.
- **Annotation validation:** each compiler-plugin rule, and the effective-operations
  intersection, including the empty case.
- **Code generation:** expected-output tests for each protocol, with single and composite keys,
  every action combination, and the default and custom base path.
- **Compilation:** the generated package compiles on every persist datastore.
- **Behaviour:** an end-to-end CRUD round trip per protocol against the in-memory and H2
  datastores, including each error category.
- **Backward compatibility:** existing persist tests pass unchanged without
  `options.dataservice`.

## Risks and Assumptions

- **Unbounded reads.** Get many returns every record until paging is added. On large tables
  this is slow and memory-heavy. The envelope lets paging be added without breaking clients,
  but until then the risk stays with the user.
- **Exposure without auth.** Until auth is added, every caller can perform every enabled
  action, and the default enables all four, including delete. Users must restrict exposure
  with `operations` and annotations, or protect the listener themselves.
- **Creates are not atomic with their read-back.** A create is followed by a separate read. A
  concurrent delete between the two can return a not-found error for a record that was
  created.
- **Name clashes in the default module.** Generated names share the user's namespace, as the
  persist client's already do. A user type named, for example, `EmployeeList` fails to
  compile alongside the generated one.
- **Assumption:** the persist client's resource names, keys and generated types are stable
  enough for the service to build on them.

## Dependencies

- `persist-tools`: options schema, CLI flags, model validation, code generation.
- `ballerina/persist` and the persist datastore clients.
- `ballerina/dataservice`: a new library.
- `ballerina/http`, `ballerina/graphql` and `ballerina/mcp`, through the protocol proposals.

## Future work

### Follow-up proposals

Each protocol proposal defines how the CRUD shape in this proposal is rendered. Each covers the
same points: the listener and service type, how each operation maps to a resource, query,
mutation or tool, how keys and types map, how the error categories map, the base path, and the
listener configuration.

| Proposal | Scope |
|---|---|
| 2. [HTTP](NNNN_data_service_http.md) | `http:Service`: collection and key paths, methods, status codes, request and response bodies |
| 3. [GraphQL](NNNN_data_service_graphql.md) | `graphql:Service`: queries and mutations, input and output types, type mappings for persist's time types and `byte[]`, error mapping |
| 4. [MCP](NNNN_data_service_mcp.md) | `mcp:StreamableHttpService`: tool names, input and output schemas, descriptions from doc comments, tool annotations such as read-only hints, error results |

### Roadmap

| Area | Adds | Fits into v1 through |
|---|---|---|
| Query options | Filtering, sorting, paging, field selection, count | The `read` action; fields added to the `<Entity>List` envelope; the reserved `$` names |
| Auth | Authentication at the listener; scopes per action in `@dataservice:Expose`; later, row-level rules | The action names; annotations that only narrow |
| Relations | Embedded reads, relation routes, nested writes | The reserved path segment after an entity's key |
| Field rules | Hidden fields and read-only fields, as field annotations | The model holding what the data allows |
| More write operations | Upsert, full replace, update or delete by filter, once the persist client supports them | The `create`, `update` and `delete` actions |
| Custom operations | Named queries exposed as operations | The reserved `EXECUTE` action |
| Several protocols | Several services from one entry, on a shared listener | `protocol` also accepting an array |
| More protocols | OData, SOAP | The reserved `odata` and `soap` values |
| API descriptions | OpenAPI, GraphQL schema, MCP tool descriptions | Generated alongside the service |
