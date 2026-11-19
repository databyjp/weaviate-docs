---
title: Interface
sidebar_position: 85
description: "RESTful, GraphQL and gRPC API interfaces with client library support for Weaviate integration."
image: og/docs/concepts.jpg
# tags: ['architecture', 'interface', 'API design']
---

You can manage and use Weaviate through its APIs. Weaviate has RESTful, GraphQL, and gRPC APIs. The client libraries broadly mirror this API surface, although feature coverage can vary by language; see the [client library pages](/weaviate/client-libraries/index.mdx) for what each one supports. Some clients, e.g. the Python client, have additional functionality, such as full schema management and batching operations. This way, Weaviate is easy to use in custom projects. Additionally, the APIs are intuitive, so it is easy to integrate into your existing data landscape.

This page explains API choices and design. Prefer current [client libraries](/weaviate/client-libraries/index.mdx) for new applications. Retained GraphQL examples describe legacy functionality only where GraphQL is enabled and available; see the [migration guide](../api/graphql/migration.mdx).

## API Design

### Design: UX & Weaviate Features

User Experience (UX) is one of our most valuable principles. Weaviate should be easy to understand, intuitive to use and valuable, desirable and usable to the community. The interaction with Weaviate is naturally very important for its UX. Weaviate's APIs are designed from the perspective of user needs, keeping the software features in mind. We do user research, user testing and prototyping to make sure all features resonate with our users. User requirements are continuously gathered during collaborative discussions. We match user needs with the functions of Weaviate. When there is a strong need from the user or application perspective, we may extend Weaviate's functions and APIs. When there is a new Weaviate function, this will naturally be accessible via (new) API functions.

The UX of Weaviate's APIs is designed following the UX Honeycomb usability rules, defined by Peter Morville.

### Choosing an API

- **Management and object CRUD**: use REST directly or through a client library.
- **Search and aggregation**: current client libraries primarily use gRPC. Feature coverage varies by language and client version.
- **HTTP-only stacks**: the REST Search API offers JSON search and aggregation endpoints, but remains **preview** and is not a promise of parity with GraphQL or every client feature. See the [API overview](../api/index.mdx) and [migration guidance](../api/graphql/migration.mdx).
- **Existing GraphQL integrations**: check deployment availability and the deprecation guidance below before relying on them.

Client libraries combine the appropriate transports behind language-specific methods; they are not limited to a REST-plus-GraphQL model. There is no general commitment to eventual parity across all APIs.

## GraphQL

import GraphQLDeprecation from '/_includes/graphql-deprecation.mdx';

<GraphQLDeprecation />

### Why GraphQL?

The following explains the historical GraphQL design rationale, not a recommendation over current clients. These examples require GraphQL to be enabled and supported by the cluster configuration:

- **Data structure**.
  - Data in Weaviate follows a class-property structure. Data objects can be queried by their class and properties with GraphQL.
  - It is possible to link data in Weaviate with cross-references. A Graph query language like GraphQL is very useful here.
- **Performance**.
  - With GraphQL, there is no over/under-fetching. You get back exactly the information about data objects that you query, nothing more and nothing less. This is beneficial for performance.
  - Reducing the number of requests. With GraphQl, you can make highly efficient and precise queries that usually require many more queries with a traditional RESTful API for the same results.
- **User Experience**
  - Reducing complexity.
  - Less error-prone (because of its typed schema)
  - Custom design
  - Data exploration and fuzzy search is possible

### GraphQL Design Principles

GraphQL queries are designed to be intuitive and fit Weaviate's features. [This article on Hackernoon](https://hackernoon.com/how-weaviates-graphql-api-was-designed-t93932tl) tells you more about how GraphQL API was designed (note that examples show an older Weaviate and GraphQL API version). The following three points are key in the design:

- **Natural language**. The GraphQL queries follow a natural language pattern as much as possible. The function of a query is easy to understand and queries are easy to write and remember. An example query where you can recognize human language is: "_Get_ the _title_ of the _Articles_ where the _wordcount_ is _greater than_ _1000_. The most important words in this query are also used in the GraphQL query:

```graphql
{
  Get {
    Article(
      where: {
        path: ["wordCount"] # Path to the property that should be used
        operator: GreaterThan # operator
        valueInt: 1000 # value (which is always = to the type of the path property)
      }
    ) {
      title
    }
  }
}
```

There are currently three main functions in a GraphQL request: "Get{}", "Explore{}" and "Aggregate{}".

- **Classes & properties**. Data in Weaviate has a class-property structure, where cross-references may appear between data object. The class name of the data to return is written one layer deeper than the 'main function'. The next layer consists of the properties and cross-reference properties to return per class:

```graphql
{
  <Function> {
      <Class> {
        <property>

        <cross_reference-property> {
            ... on <ClassOfBeacon> {
                <property>
            }
        }

        _<additional-property> {
            <additional-field>
        }
      }
  }
}
```

- **Query filters (search arguments) dependent on database setup**. You can add filters on class level to filter objects. Scalar (`where` filters) can be combined with vector (`near<...>`) filters. Depending on your Weaviate setup (which modules you have connected), additional filters may be used. A filter can look like (using the [`qna-transformers` module](/weaviate/modules/qna-transformers.md)):

```graphql
{
  Get {
    Article(
      ask: {
        question: "Who is the king of the Netherlands?"
        properties: ["summary"]
      }
      limit: 1
    ) {
      title
      _additional {
        answer {
          result
        }
      }
    }
  }
}
```

### GraphQL Design of Main Functions

1. **Data search: `Get {}`**: to search for data objects when you know the class name of the data objects you're looking for.
2. **Explorative & fuzzy search: `Explore {}`**: to search in a fuzzy way, when you don't know the data schema and class names.
3. **Data analysis (meta data): `Aggregate {}`**: to search for meta data, and do data analysis of data aggregations.

## gRPC API support

Alongside the RESTful and GraphQL APIs, Weaviate serves a gRPC API. gRPC uses HTTP/2 and Protocol Buffers, which can reduce serialization and transport overhead compared with equivalent JSON requests. End-to-end performance still depends on the operation, payload, network, and workload. It was introduced in Weaviate `v1.19.0` and has been considered stable since `v1.23.7`.

gRPC carries most of the search and batch import traffic that the client libraries generate, so a Weaviate deployment usually exposes a gRPC port (`50051` by default, configurable with the `GRPC_PORT` [environment variable](/deploy/configuration/env-vars/index.md)) in addition to the REST port. Client coverage is not uniform: the [Python](/weaviate/client-libraries/python/index.mdx), [TypeScript](/weaviate/client-libraries/typescript/index.mdx), [Java](/weaviate/client-libraries/java/index.mdx), and [C#](/weaviate/client-libraries/csharp.mdx) clients use gRPC for queries and batch operations. The [Go](/weaviate/client-libraries/go/index.md) client uses gRPC for queries and batch imports. The [Go client `v5`](/weaviate/client-libraries/go/go-v5.md) uses it for batch imports and reaches gRPC search through its experimental API.

For the Protobuf definitions and for ways to call the API without a client library, see the [gRPC API reference](../api/grpc.md).

Browser and compatible edge applications can use the [TypeScript web client](../client-libraries/typescript/web-client.mdx) over [gRPC-Web](../api/grpc.md#grpc-web), available from v1.38.3 on the REST port rather than the native gRPC port. Its supported operations differ from native clients; in particular, server-side batching is unavailable.

## MCP interface

Weaviate also includes an MCP server for compatible AI-agent clients, using Streamable HTTP at `/v1/mcp` on the REST endpoint. Collection discovery, searches, and optional writes depend on configuration and permissions. See [MCP configuration](../configuration/mcp-server.mdx) for availability and access controls; this is not an API-parity or universal-default guarantee.

## Weaviate Console

The [Weaviate Console](/go/console?utm_content=others) is a dashboard to manage Weaviate clusters from WCD, and access Weaviate instances running elsewhere. Where GraphQL is enabled and available, the Query Module can make GraphQL queries. The screenshot below illustrates that legacy query interface, not universal current cluster availability.

![GraphQL Query Module in Weaviate Console](./img/console-capture.png)

## Weaviate Clients

Weaviate has several client libraries: in [C#](/weaviate/client-libraries/csharp.mdx), [Go](/weaviate/client-libraries/go/index.md), [Java](/weaviate/client-libraries/java/index.mdx), [Python](/weaviate/client-libraries/python/index.mdx), and [TypeScript/JavaScript](/weaviate/client-libraries/typescript/index.mdx). The client libraries broadly mirror the server API surface, although feature coverage varies by language. See the [client library pages](/weaviate/client-libraries/index.mdx) for what each one supports. Some clients, e.g. the Python client, have additional functionality, such as full schema management and batching operations. This way, Weaviate is easy to use in custom projects. The APIs are intuitive to use, so it is easy to integrate Weaviate into your existing data landscape.

## Further resources

:::info Related pages

- [References: GraphQL API](../api/graphql/index.md)
- [References: RESTful API](/weaviate/api/rest).
- [References: Client Libraries](../client-libraries/index.mdx).
  :::

## Questions and feedback

import DocsFeedback from '/\_includes/docs-feedback.mdx';

<DocsFeedback/>
