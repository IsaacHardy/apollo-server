---
title: Mocking
description: Mock your GraphQL data based on a schema.
---

The strongly-typed nature of a GraphQL API lends itself extremely well to mocking. This is an important part of a GraphQL-First development process, because it enables frontend developers to build out UI components and features without having to wait for a backend implementation.

Even when the UI is already built, it can let you test your UI without waiting on slow database requests, or build out a component harness using a tool like React Storybook without needing to start a real GraphQL server.

## Default mock example

This example demonstrates mocking a GraphQL schema with just one line of code, using `apollo-server`'s default mocking logic.

```js
const { ApolloServer } = require('apollo-server');

const typeDefs = `
type Query {
  hello: String
}
`;

const server = new ApolloServer({
  typeDefs,
  mocks: true,
});

server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`)
});
```

> Note: If `typeDefs` has custom scalar types, `resolvers` must still contain the `serialize`, `parseValue`, and `parseLiteral` functions

Mocking logic simply looks at the type definitions and returns a string where a string is expected, a number for a number, etc. This provides the right shape of result. For more sophisticated testing, mocks can be customized them to a particular data model.

## Customizing mocks

In addition to a boolean, `mocks` can be an object that describes custom mocking logic, which is structured similarly to `resolvers` with a few extra features aimed at mocking. Namely `mocks` accepts functions for specific types in the schema that are called when that type is expected. The functions in `mocks` would be used when no resolver in `resolvers` is specified. In this example `hello` will return `'Hello'` and `resolved` will return `'Resolved'`.

```js line=16-20
const { ApolloServer } = require('apollo-server');

const typeDefs = `
type Query {
  hello: String
  resolved: String
}
`;

const resolvers = {
  Query: {
    resovled: () => 'Resolved',
  },
};

const mocks = {
  Int: () => 6,
  Float: () => 22.1,
  String: () => 'Hello',
};

const server = new ApolloServer({
  typeDefs,
  resolvers,
  mocks,
});

server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`)
});
```

Similarly to `resolvers`, `mocks` allows the description of object types with the fields. Take note that the value corresponding to `Person` is a function that returns an object that contains fields pointing at more functions:

```js
const mocks = {
  Person: () => ({
    name: casual.name,
    age: () => casual.integer(0, 120),
  }),
};
```

The previous example uses [casual](https://github.com/boo1ean/casual), a fake data generator for JavaScript, which returns a different result every time the field is called. In other scenarios, such as testing, a collection of fake objects or a generator that always uses a consistent seed are often necessary to provide consistent data.

### Using `MockList` in resolvers

To automate mocking a list, return an instance of `MockList`:

```js
const { MockList } = require('apollo-server');

const mocks = {
  Person: () => ({
    // a list of length between 2 and 6 (inclusive)
    friends: () => new MockList([2,6]),
    // a list of three lists each with two items: [[1, 1], [2, 2], [3, 3]]
    listOfLists: () => new MockList(3, () => new MockList(2)),
  }),
};
```

In more complex schemas, `MockList` is helpful for randomizing the number of entries returned in lists.

For example, this schema:

```graphql
type Query {
  people: [Person]
}

type Person {
  name: String
  age: Int
}
```

By default, the `people` field will always return 2 entries. To change this, we can add a mock resolver that returns `MockList`:

```js
const mocks = {
  Query: () =>({
    people: () => new MockList([0, 12]),
  }),
};
```

Now the mock data will contain between zero and 12 summary entries.

### Accessing arguments in mock resolvers

The mock functions on fields are actually just GraphQL resolvers, which can use arguments and `context`:

```js
const mocks = {
  Person: () => ({
    // the number of friends in the list now depends on numPages
    paginatedFriends: (root, args, context, info) => new MockList(args.numPages * PAGE_SIZE),
  }),
};
```

For some more background and flavor on this approach, read the ["Mocking your server with one line of code"](https://medium.com/apollo-stack/mocking-your-server-with-just-one-line-of-code-692feda6e9cd) article on the Apollo blog.

## Mocking a schema using introspection

The GraphQL specification allows clients to introspect the schema with a [special set of types and fields](https://facebook.github.io/graphql/#sec-Introspection) that every schema must include. The results of a [standard introspection query](https://github.com/graphql/graphql-js/blob/master/src/utilities/introspectionQuery.js) can be used to generate an instance of GraphQLSchema which can be mocked as explained above.

This helps when you need to mock a schema defined in a language other than JS, for example Go, Ruby, or Python.

To convert an [introspection query](https://github.com/graphql/graphql-js/blob/master/src/utilities/introspectionQuery.js) result to a `GraphQLSchema` object, you can use the `buildClientSchema` utility from the `graphql` package.

```js
const { buildClientSchema } = require('graphql');
const introspectionResult = require('schema.json');
const { ApolloServer } = require('apollo-server');

const schema = buildClientSchema(introspectionResult);

const server = new ApolloServer({
  schema,
  mocks: true,
});

server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`)
});
```

## API

Under the hood, Apollo Sever uses a library for building GraphQL servers, called `graphql-tools`. The mocking functionality is provided by the function [`addMockFunctionsToSchema`]() and [`MockList`]() is exported directly.

TODO point to the graphql-tools api