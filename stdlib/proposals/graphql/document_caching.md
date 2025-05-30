# GraphQL Document Caching

- Author: @DimuthuMadushan
- Reviewers: @daneshk @ThisaruGuruge
- Created: 2025-03-28
- Updated: 2025-03-28
- Issue: [#1345](https://github.com/ballerina-platform/ballerina-spec/issues/1345)
- Status: Submitted

## Summary

This proposal is to introduce GraphQL document caching. The GraphQL document caching reduces the document parsing overhead by skip the parsing of the same document, which leads to improved performance.

## Goals

* Enhance GraphQL server performance by avoiding redundant parsing of the same document.

## Non-Goals

* Add Support for GraphQL persisted queries.

## Motivation

GraphQL document parsing takes around 40% of the total processing time which impacts performance. Customers have requested improvements in this area, and other implementations, like Apollo Server’s [DocumentStore](https://www.apollographql.com/docs/apollo-server/api/apollo-server#documentstore), already offer similar functionality. By introducing in-memory document caching, it can be avoided parsing the same query multiple times, making the server more efficient. This will help users who frequently reuse queries, improving overall GraphQL server performance.

## Description

This proposal focuses on GraphQL document caching on the server side, which can be enabled by service  configuration. The Ballerina cache module manages the cache internally using a single in-memory cache table. The GraphQL engine automatically generates cache keys by hashing the document and caches the parsed syntax tree of the GraphQL document. It uses an LRU eviction policy, and cache entries remain valid until they are evicted by the LRU policy. Once the cache hits, it skips the document parsing process.

> **Note**: Documents are cached only if they are parsed successfully without errors.

![Image](https://github.com/user-attachments/assets/84ca81ba-6387-4a26-b29d-13e5bc8e6a26)

### Enable Document Cache

At the service level, the users can provide the document cache configurations. These configurations are introduced as a separate field named **documentCacheConfig** in the `graphql:ServiceConfig` annotation. This field is optional and is of type **DocumentCacheConfig** record.

```ballerina
public type DocumentCacheConfig record {|
    boolean enabled = true;
    int maxSize = 100;
|};
```

 * The `enabled` field  - State whether the document cache is enabled or disabled, with the default value set to true.
 * The `maxSize` field - Defines the maximum capacity of the cache table in terms of entries, with the default size set to 100 entries.
    > **Note**:The average size of a complex GraphQL document is typically around 10KB. The default value is set to 100 to ensure the document cache size remains at approximately 1MB. This value can be adjusted according to the user's requirements.

### Cache Key Generation

To generate a cache key, a unique value is needed for each document. For GraphQL documents, the document itself provides this unique value. Since documents can be in different formats, standardizing the document is essential for consistent cache key generation. The cache key is generated by converting the standardized document into a Base64-encoded hash value. The `MD5` algorithm is used to generate the hash key, considering its performance benefits and the fact that the cache key is not used for security-sensitive purposes.

#### Document Standardizing
Usually, GraphQL documents are sent as indented, human-readable text with new lines, spaces, and comments, even though a valid GraphQL document does not necessarily include all of these. The document standardization process involves ignoring whitespaces, new lines, and comments to achieve the following, as it is used to generate the cache key.
 * **Consistency**: By removing whitespace, comments and formatting, the same query will always generate the same key.
 * **Query Uniqueness**: Ensures that different queries get different keys.

Therefore, the GraphQL documents are standardized using the following rules.
1. Remove all whitespaces.
2. Remove all new lines.
3. Remove all comments.
4. Separate each field with a comma.

###### Example: Document standardizing

Before:
```graphql
query getDetail($userID:Int! $searchTerm:String! $user: User!){
  #retrieve user details
  user(id: $userId) {
     id
     name
     email
     role
  }
  #retrieve post details
  posts {
     id
     title
  }
  search(query: $searchTerm) { 
     ... on User {
         id
         name
     } 
     ... on Post {
         id
         title
     } 
  }
  usersByIds(input: $user) {
         id
         name
         email
   }
}
 ```

After
```graphql
 query,getDetail($userID:Int!$searchTerm:String!$user:User!){user(id:$userId){id,name,email,role},posts{id,title},search(query:$searchTerm){...on User{id,name},...on Post{id,title}},usersByIds(input:$user){id,name,email}}
``` 

### Recommended Best Practices
 * If a query includes literal values as arguments, those literals become part of the cached document. As a result, similar requests with different argument values will be treated as separate entries, requiring parsing and caching each time. To maximize cache efficiency, it is recommended to pass arguments using variables instead of literals within the document.
 
###### Example: Recommended way
```ballerina
query GetProfile($teacherName:String!, $studentName: String!) {
    teacher(name: $teacherName) {
      id
    }
    student(name: $studentName) {
      id
    }
}
```

###### Example: Non-Recommended way 
```ballerina
query GetProfile {
    teacher(name: "Bob") {
      id
    }
    student(name: "Jessie") {
      id
    }
}
```

 * Cache eviction follows the LRU policy. Ensure that the cache's maxSize is configured appropriately based on available memory since this does not cover memory limiting for the cache.

### Future Plans
 * Standardize documents before parsing
 
   To minimize the overhead of the parsing process, document standardization can be applied before parsing. This step removes unnecessary details from the document while preserving its validity as a GraphQL document. However, **a proper performance analysis should be conducted to evaluate the potential benefits of this approach.**

* Provide support for configuring document cache maxSize in terms of memory

   Many document caching implementations allow configuring the cache's maximum size in terms of memory. Currently, the Ballerina cache package only supports size configuration by entries. Once memory-based size configuration is added, it can be used for GraphQL document caching.

* Implement maxQuerySize configuration to limit the size of a GraphQL document

   Introducing a configuration to limit the maximum size of a GraphQL document will help reduce the risk of storing large queries in memory, preventing memory usage issues in the cache. This will ensure that queries are appropriately sized and managed within the in-memory cache.

### Example: Document Caching
```ballerina
import ballerina/graphql;

public type Profile record {|
    string name;
    int age;
|};

@graphql:ServiceConfig {
    documentCacheConfig: {
        maxSize: 20
    }
}
service /graphql on new graphql:Listener(9090) {
    resource function get profile() returns Profile {
        return {
            name: "Walter White",
            age: 51
        };
    }
}
```
