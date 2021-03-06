# `@grafoo/react`

<p><i>Grafoo React Bindings</i></p>

<p>
  <a href=https://circleci.com/gh/grafoojs/grafoo>
    <img
      src=https://img.shields.io/circleci/project/github/grafoojs/grafoo/master.svg?label=build
      alt=build
    />
  </a>
  <a href=https://codecov.io/github/grafoojs/grafoo>
    <img
      src=https://img.shields.io/codecov/c/github/grafoojs/grafoo/master.svg
      alt="coverage"
    />
  </a>
  <a href=https://github.com/grafoojs/grafoo>
    <img
      src=https://img.shields.io/npm/v/@grafoo/bindings.svg
      alt=npm
    >
  </a>
  <a href=https://www.npmjs.com/package/@grafoo/react>
    <img
      src=https://img.shields.io/npm/dm/@grafoo/bindings.svg
      alt=downloads
    >
  </a>
  <a href=https://www.npmjs.com/package/@grafoo/react>
    <img
      src=https://img.shields.io/bundlephobia/minzip/@grafoo/react.svg?label=size
      alt=size
    >
  </a>
  <a href=https://prettier.io>
    <img
      src=https://img.shields.io/badge/code_style-prettier-ff69b4.svg
      alt="code style: prettier"
    />
  </a>
  <a href=https://lernajs.io>
    <img
      src=https://img.shields.io/badge/maintained%20with-lerna-cc00ff.svg
      alt="mantained with: lerna"
    />
  </a>
</p>

## Install

```
$ npm i @grafoo/{core,react} && npm i -D @grafoo/babel-plugin
```

## Setup

Assuming you already have babel installed, the only additional step required to build an application with Grafoo is to configure [`@grafoo/babel-plugin`](https://github.com/grafoojs/grafoo/tree/master/packages/babel-plugin). The options it accepts are `idFields` - the fields Grafoo will take to build unique identifiers, and `schema`, which is a relative path to your schema file.

```json
{
  "plugins": [
    [
      "@grafoo/babel-plugin",
      {
        "schema": "schema.graphql",
        "idFields": ["id"]
      }
    ]
  ]
}
```

## API

### `Provider`

`Provider` receives a single `client` instance prop that will be consumed by the `Consumer` components.

```jsx
import React from "react";
import createClient from "@grafoo/core";
import { Provider } from "@grafoo/react";

function fetchQuery(query, variables) {
  const init = {
    method: "POST",
    body: JSON.stringify({ query, variables }),
    headers: {
      "content-type": "application/json"
    }
  };

  return fetch("http://some.graphql.api", init).then(res => res.json());
}

const client = createClient(fetchQuery);

export default function App() {
  return (
    <Provider client={client}>
      <SomeComponent />
    </Provider>
  );
}
```

### `Consumer`

`Consumer` is the component that performs query requests to your GraphQL API. It accepts the following props:

| Name      | Type     | Default | Required | Descrition                                                   |
| --------- | -------- | ------- | -------- | ------------------------------------------------------------ |
| query     | object   | -       | false    | a query created with `@grafoo/core/tag`                      |
| variables | object   | -       | false    | a GraphQL variables object for the `query` prop              |
| mutations | object   | -       | false    | a map of mutations (description below)                       |
| skip      | boolean  | false   | false    | whether `Consumer` should skip the `query` request initially |
| children  | function | -       | false    | a render function (description below)                        |

### Render parameter

The `Consumer` render function takes as parameter an object with the following props:

| Name    | type     | Descrition                                                   |
| ------- | -------- | ------------------------------------------------------------ |
| client  | object   | the client instance                                          |
| load    | function | a method to execute a query with the `query` prop            |
| loading | boolean  | whether the client is executing a query or not               |
| loaded  | boolean  | whether the query data is already cached                     |
| errors  | string[] | an array of GraphQL errors from a failed request to your API |

The remaining props are:

- the data fetched by the client and shaped according to your `query`
- mutation functions generated by the `mutations` object prop

#### Example

```jsx
import React from "react";
import gql from "@grafoo/core/tag";
import { Consumer } from "@grafoo/react";

const ALL_POSTS = gql`
  query getPosts($orderBy: PostOrderBy) {
    allPosts(orderBy: $orderBy) {
      title
      content
      createdAt
      updatedAt
    }
  }
`;

export default function Posts() {
  return (
    <Consumer query={ALL_POSTS} variables={{ orderBy: "createdAt_DESC" }}>
      {({ client, load, loaded, loading, errors, allPosts }) => (
        <h1>
          <marquee>👆 do whatever you want with the variables above 👆</marquee>
        </h1>
      )}
    </Consumer>
  );
}
```

### Mutations

The `mutations` prop is a map of _mutation objects_ that are shaped like so:

```js
const createPost = {
  query: CREATE_POST_MUTATION,
  optimisticUpdate: ({ allPosts }, variables) => ({
    allPosts: [{ ...variables.postInput, id: "tempID" }, ...allPosts]
  }),
  update: ({ allPosts }, response) => ({
    allPosts: allPosts.map(p => (p.id === "tempID" ? response.post : p))
  })
};

const mutations = { createPost };
```

A mutation object receives the following props:

| Name             | Type     | Required | Descrition                                                          |
| ---------------- | -------- | -------- | ------------------------------------------------------------------- |
| query            | object   | true     | a mutation query created with `@grafoo/core/tag`                    |
| update           | function | false    | updates the cache when a request is completed (description below)   |
| optimisticUpdate | function | false    | updates the cache before a request is completed (description below) |

Each mutation will generate a single function that accepts a GraphQL variables object as argument and return a promise that will resolve with the mutation data or reject with GraphQL `errors`.

```ts
type MutationFn = (variables: Variables) => Promise<MutationResponse>;
```

### Mutation query dependency

**Important** to notice that to update the cache `update` and `optimistUpdate` hooks depend on a `query` and it's `variables` object props (so they need be declared in the `Consumer`). If you need to perform a mutation but updating the cache is not strictly important you can just use the mutation promise resolved data or use the client instance directly.

### `update`

```ts
type UpdateFn = (query: QueryData, response: MutationResponse) => CacheUpdate;
```

The mutation `update` function is resposible to update the cache when the request is completed. It receives as paremeters an object containing the data from the query it depends upon and the mutation response sent by the server. `update` return type is an object that describes the changes to be made to the cache.

### `optimisticUpdate`

```ts
type OptimistcUpdateFn = (query: QueryData, variables: Variables) => CacheUpdate;
```

In modern UIs it's to be expected that every user interaction occur in a fraction of seconds. `optimisticUpdate` responsability is to skip the mutation network roundtrip and update the cache instantaneously, making sure such interactions are as fast as they can be. `optimisticUpdate` as in `update` takes as first paremater the depedent query data. As second paremater it receives the variables object with which it's correpondent generated mutation function was called. And again it should return an object that describes the changes to be made to cache.

If you want to perform an optimitic update you have to make sure that the data you are inserting contains the field or fields to extract a unique identifier. For instance, say `@grafoo/babel-plugin` `idFields` option is set to insert a property `id`. Is to be expected that your update has that field mocked.

#### Example

```jsx
import React from "react";
import gql from "@grafoo/core/tag";
import { Consumer } from "@grafoo/react";

const ALL_POSTS = gql`
  query getPosts($orderBy: PostOrderBy) {
    allPosts(orderBy: $orderBy) {
      title
      content
      createdAt
      updatedAt
    }
  }
`;

const CREATE_POST = gql`
  mutation createPost($content: String, $title: String, $authorId: ID) {
    createPost(content: $content, title: $title, authorId: $authorId) {
      title
      content
      createdAt
      updatedAt
    }
  }
`;

const mutations = {
  createPost: {
    query: CREATE_POST,
    optimisticUpdate: ({ allPosts }, variables) => ({
      allPosts: [{ ...variables, id: "tempID" }, ...allPosts]
    }),
    update: ({ allPosts }, data) => ({
      allPosts: allPosts.map(p => (p.id === "tempID" ? data.createPost : p))
    })
  }
};

const submit = mutate => event => {
  event.preventDefault();

  const { title, content } = event.target.elements;

  mutate({ title: title.value, content: content.value }).then(mutationData => {
    console.log(mutationData);
  });
};

export default function PostForm() {
  return (
    <Consumer query={ALL_POSTS} mutations={mutations}>
      {({ createPost }) => (
        <form onSubmit={submit(createPost)}>
          <input name="title" />
          <textarea name="content" />
          <button>submit</button>
        </form>
      )}
    </Consumer>
  );
}
```

## LICENSE

[MIT](https://github.com/grafoojs/grafoo/blob/master/LICENSE)
