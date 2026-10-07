# Data service generation for bal persist: GraphQL

- Author: @niveathika
- Reviewers: 
- Created: 2026-10-07
- Updated: 2026-10-07
- Issue: TBD
- Status: Draft

> This file is renamed to its issue number once the issue is created.

## Summary

Defines how the protocol-neutral CRUD shape in
[Data service generation for bal persist](1503_data_service.md) is rendered as a
`graphql:Service` when `options.dataservice.protocol = "graphql"`.

Reads become `Query` fields and writes become `Mutation` fields. Unlike HTTP and MCP, the
GraphQL service cannot use the persist-generated types directly, so the tool generates GraphQL
input and output types and converts between them and the persist types.

Everything marked ✅ was compiled and run on Ballerina 2201.13.3 with `ballerina/graphql`
1.18.2 and a generated persist in-memory client.

## Goals

- Define the query and mutation fields, their arguments and their result types.
- Define the GraphQL types generated for each entity, and the type mappings they need.
- Map the error categories to GraphQL errors.

## Non-goals

- Query options, relations and auth. They follow the roadmap in #1503. Relations will make
  GraphQL types into service classes; this proposal keeps them records.
- Subscriptions.
- Relay-style connections. Get many returns the `<Entity>List` envelope from #1503.

## Design

### Listener and service

```ballerina
configurable int dataservicePort = 9090;

final Client dataserviceClient = check new ();
listener http:Listener dataserviceListener = new (dataservicePort);
listener graphql:Listener dataserviceGraphqlListener = new (dataserviceListener);

service /graphql on dataserviceGraphqlListener {
    // query and mutation fields for every exposed entity
}
```

- **Base path:** `options.dataservice.basePath`, `/graphql` by default.
- **Shared listener:** the GraphQL listener is built on a module-level `http:Listener`, so a
  user-written HTTP service can attach to `dataserviceListener` and share the port. ✅

### Operations

For an entity `Employee` with key `id`:

| Operation | Action | Field | Result |
|---|---|---|---|
| Get many | `read` | `Query.employees` | `EmployeeRecordList!` |
| Get one | `read` | `Query.employee(id: Int!)` | `EmployeeRecord` (null when no record has the key) |
| Create one | `create` | `Mutation.createEmployee(input: CreateEmployeeInput!)` | `EmployeeRecord!` |
| Create many | `create` | `Mutation.createEmployees(input: [CreateEmployeeInput!]!)` | `EmployeeRecordList!` |
| Patch one | `update` | `Mutation.updateEmployee(id: Int!, input: UpdateEmployeeInput!)` | `EmployeeRecord!` |
| Delete one | `delete` | `Mutation.deleteEmployee(id: Int!)` | `EmployeeRecord!` |

- **Field names:** get many uses the persist resource name (`employees`); every other field uses
  the entity name in lower camel case (`employee`, `createEmployee`).
- **Get one returns null for a missing key**, rather than an error. This is the usual GraphQL
  convention for a lookup. ✅
- **Composite keys:** one argument per key field, in declaration order, for example
  `assignment(employeeId: Int!, projectCode: String!)`.
- **Disabled operations** are not generated, so the field is absent from the schema and a
  query naming it fails validation.

#### Generated service

```ballerina
service /graphql on dataserviceGraphqlListener {

    resource function get employees() returns EmployeeRecordList|error {
        EmployeeRecord[] items = check from Employee e in dataserviceClient->/employees(Employee)
            select toEmployeeRecord(e);
        return {items};
    }

    resource function get employee(int id) returns EmployeeRecord?|error {
        Employee|persist:Error e = dataserviceClient->/employees/[id];
        if e is persist:NotFoundError {
            return ();
        }
        return toEmployeeRecord(check e);
    }

    remote function createEmployee(CreateEmployeeInput input) returns EmployeeRecord|error { ... }

    remote function createEmployees(CreateEmployeeInput[] input) returns EmployeeRecordList|error { ... }

    remote function updateEmployee(int id, UpdateEmployeeInput input) returns EmployeeRecord|error { ... }

    remote function deleteEmployee(int id) returns EmployeeRecord|error { ... }
}
```

### Types

#### Why the persist types cannot be used

The probe found three rules in `ballerina/graphql` that the persist types break: ✅

| Persist type | Rule it breaks |
|---|---|
| `EmployeeInsert` | It is a type alias (`type EmployeeInsert Employee;`), and a type alias for a record is not supported as a GraphQL type |
| `Employee` as output with `EmployeeInsert` as input | The same type cannot be both an input and an output type |
| `time:Date` and the other `time:*` records | They contain `time:Seconds`, an alias for `decimal`, which is not supported; and as records they would also be used as both input and output |

So the tool generates three records per entity:

```ballerina
type EmployeeRecord record {|
    readonly int id;          // key fields keep readonly
    string name;
    string email;
    decimal salary;
    string hiredOn;           // time:Date as an ISO 8601 date
    string? phone;
|};

type CreateEmployeeInput record {|
    int id;
    string name;
    string email;
    decimal salary;
    string hiredOn;
    string? phone;
|};

type UpdateEmployeeInput record {|
    string name?;
    string email?;
    decimal salary?;
    string hiredOn?;
    string? phone?;
|};
```

plus the list envelope `EmployeeRecordList {EmployeeRecord[] items;}` and the conversion
functions between these and `Employee`, `EmployeeInsert` and `EmployeeUpdate`.

- **Output and input must differ structurally.** Ballerina types are structural, so an input
  record with the same fields as the output record is treated as the same type and rejected.
  ✅ The output record keeps `readonly` on its key fields and the input record does not, which
  makes them distinct for every entity, including one whose insert type has the same fields as
  the entity.
- **Names:** the GraphQL type name is the Ballerina type name, and `ballerina/graphql` has no
  annotation to rename a type. The output type cannot be named `Employee`, because the persist
  type of that name is in the same module. See open question 1.

#### Type mappings

| Persist field type | GraphQL type | Notes |
|---|---|---|
| `int` | `Int` | |
| `float` | `Float` | |
| `decimal` | `Decimal` | ✅ |
| `string` | `String` | |
| `boolean` | `Boolean` | |
| `T?` | nullable `T` | |
| enum | enum | |
| `byte[]` | `String` | base64 |
| `time:Date` | `String` | ISO 8601 date, `2024-01-02` ✅ |
| `time:TimeOfDay` | `String` | ISO 8601 time |
| `time:Civil` | `String` | ISO 8601 date-time without offset |
| `time:Utc` | `String` | ISO 8601 date-time with `Z` |

`ballerina/graphql` has no custom scalars, so date and time values are strings. A string that
does not parse as the expected format is an invalid-input error.

#### Patch with nullable fields

In `UpdateEmployeeInput`, a field that is absent means "do not change" and a field set to
`null` means "set to null". The conversion checks `hasKey` for nullable fields so the two stay
distinct.

### Errors

GraphQL reports errors in the response's `errors` array, with `data` for the failed field set
to `null`. ✅

| Category | GraphQL result |
|---|---|
| Not found, on get one | `null`, no error ✅ |
| Not found, on patch or delete | an error with the persist message ✅ |
| Conflict | an error with the persist message ✅ |
| Constraint violation | an error with the persist message |
| Invalid input | a validation error from `ballerina/graphql`, naming each missing or mistyped field, before the resolver runs ✅ |
| Internal | an error with a generic message; the detail is logged |

Example: ✅

```json
{"errors": [{"message": "A record with the key '1' already exists for the entity 'Employee'.",
             "locations": [{"line": 1, "column": 12}], "path": ["createEmployee"]}],
 "data": null}
```

The error category is only in the message text. See open question 2.

### Configuration

| Setting | Where | Default |
|---|---|---|
| Base path | `options.dataservice.basePath` | `/graphql` |
| Port | `dataservicePort` in `Config.toml` | `9090` |

## Alternatives

- **Use the persist types and skip the conversion.** Not possible: see "Why the persist types
  cannot be used".
- **Return an error for get one with a missing key.** Rejected: GraphQL clients expect a
  nullable lookup.
- **Service classes for output types.** Needed once relations are added, because a relation
  field is resolved lazily. Not needed for this proposal.
- **Mark keys with `@graphql:ID`.** That makes keys GraphQL `ID`s, serialized as strings.
  Plain scalar keys are kept so that `Int` keys stay numbers.

## Testing

- Code generation: expected output for every persist field type in the mapping table, single
  and composite keys, and every action combination.
- Schema: an introspection query lists exactly the expected fields and argument types for each
  action combination.
- Behaviour: every row of the operations and errors tables, including absent versus `null`
  fields in a patch.
- Round trip of every date and time type through create and get one.

## Risks and Assumptions

- **More generated code than HTTP or MCP.** Three records and their conversions per entity. A
  bug in the conversion changes data, so the round-trip tests matter.
- **Error categories are not machine-readable** until open question 2 is settled.

## Open questions

1. **Output type names.** `EmployeeRecord` is used here because `Employee` is taken by the
   persist type in the same module. Alternatives: rename only on a clash, or ask
   `ballerina/graphql` for a type-name annotation.
2. **Error codes.** Can the category be put in each error's `extensions` (for example
   `{"code": "NOT_FOUND"}`), as most GraphQL servers do? This needs checking against
   `ballerina/graphql`.
3. **Time types in the other protocols.** GraphQL has to use ISO 8601 strings. HTTP and MCP
   could do the same for consistency; see the HTTP proposal.

## Dependencies

- [Data service generation for bal persist](1503_data_service.md)
- `ballerina/graphql`

## Future work

- Relations as object fields, with service classes for output types.
- Query options as field arguments.
- Error extensions, once open question 2 is settled.
