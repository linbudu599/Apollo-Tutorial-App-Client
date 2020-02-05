# Apollo-Tutorial-App-Client

> Apollo 实践 全栈 App 客户端部分
> Typescript + React-Apollo + React
> 本篇文章分为两个部分，使用`Apollo-Client`来连接 React 应用，与使用`React-Apollo`来搭建客户端

## Apollo-Client

Apollo-Client（以下简称 AC 好了）是一个数据流管理方案，就像 `Redux` 一样，它并不关心你的视图层是用哪个框架实现的，即使是原生 JS 也没问题。
得益于它的缓存机制，AC 能够为所有本地和远程数据提供专一的来源（store？）。  
（上面这段话是我翻译的，来自于官方文档）

知道了 AC 的作用，那么想要了解如何使用就简单的多了，因为其实都逃不开查询和变更操作，而 AC 的优势就是它的缓存机制，官网上说是 `intelligent cache`，得劲。

需要安装额外的包，包括：

- `apollo-client`

- `react-apollo`

- `graphql-tag`，提供 `gql` 方法，包裹 graphql 查询语句，并将其解析为 AST

### 创建一个简易的客户端

```typescript
import { ApolloClient } from "apollo-client";
import { InMemoryCache, NormalizedCacheObject } from "apollo-cache-inmemory";
import { HttpLink } from "apollo-link-http";
import { ApolloProvider } from "@apollo/react-hooks";
import gql from "graphql-tag";
import { resolvers, typeDefs } from "./resolvers";

const cache = new InMemoryCache();

const client: ApolloClient<NormalizedCacheObject> = new ApolloClient({
  cache,
  link: new HttpLink({
    // 前面的服务端地址
    uri: "http://localhost:4000",
    headers: {
      authorization: localStorage.getItem("token")
    }
  }),
  resolvers,
  typeDefs
});

ReactDOM.render(
  <ApolloProvider client={client}>// ...</ApolloProvider>,
  document.getElementById("root")
);
```

解析：

- 实例化 `ApolloClient` 类，并传入缓存、链接、类型定义和解析器，同时传入对应的配置。

- 使用 `<ApolloProvider>` 组件作为一个 `wrapper` HOC，将实例化的 client 作为属性注入。

### 进行简单的查询/写入

事实上在创建客户端之后，我们就可以直接对客户端进行一些操作。

```typescript
client
  .query({
    query: gql`
      query GetLaunch {
        launch(id: 56) {
          id
          mission {
            name
          }
        }
      }
    `
  })
  .then(result => console.log(result));

// 在缓存中写入数据
cache.writeData({
  data: {
    isLoggedIn: !!localStorage.getItem("token"),
    cartItems: []
  }
});
```

## useQuery

是的，这熟悉的命名风格，这其实就是一个自定义的 hooks，有点像我之前写过的 `useAxios`，我们传入 graphql 语句（记得用`gql`包裹），然后得到数据、加载状态、错误与一个用于取得更多结果的函数（如果你对数据做了分页处理的话，这个函数是很有必要的）。如

```typescript
const { data, loading, error, fetchMore } = useQuery<
  GetLaunchListTypes.GetLaunchList,
  GetLaunchListTypes.GetLaunchListVariables
>(GET_LAUNCHES);
```

**Tips!** 这里注入给 `useQuery` 的泛型参数并不用自己写，你可以通过运行 `npm run codegen` 来使用 apollo 内置的工具自动生成类型声明。

对于有分页需求的数据，整个应用的逻辑复杂度可能会稍微上升，如本例的完整代码：

```typescript
import React from "react";
import { useQuery } from "@apollo/react-hooks";
import gql from "graphql-tag";

import { LaunchTile, Header, Button, Loading } from "../components";
import { RouteComponentProps } from "@reach/router";
import * as GetLaunchListTypes from "./__generated__/GetLaunchList";

export const LAUNCH_TILE_DATA = gql`
  fragment LaunchTile on Launch {
    id
    isBooked
    rocket {
      id
      name
    }
    mission {
      name
      missionPatch
    }
  }
`;

const GET_LAUNCHES = gql`
  query launchList($after: String) {
    launches(after: $after) {
      cursor
      hasMore
      launches {
        ...LaunchTile
      }
    }
  }
  # import fragment
  ${LAUNCH_TILE_DATA}
`;

interface LaunchesProps extends RouteComponentProps {}

const Launches: React.FC<LaunchesProps> = () => {
  const { data, loading, error, fetchMore } = useQuery<
    GetLaunchListTypes.GetLaunchList,
    GetLaunchListTypes.GetLaunchListVariables
  >(GET_LAUNCHES);

  if (loading) return <Loading />;
  if (error || !data) return <p>ERROR</p>;

  return (
    <>
      <Header />
      {data?.launches?.launches.map((launch: any) => (
        <LaunchTile key={launch.id} launch={launch} />
      ))}
      {data.launches && data.launches.hasMore && (
        <Button
          onClick={() =>
            fetchMore({
              variables: {
                after: data.launches.cursor
              },
              updateQuery: (prev, { fetchMoreResult, ...rest }) => {
                if (!fetchMoreResult) return prev;
                // TODO: Deal With It
                return {
                  ...fetchMoreResult,
                  launches: {
                    ...fetchMoreResult.launches,
                    launches: [
                      ...prev.launches.launches,
                      ...fetchMoreResult.launches.launches
                    ]
                  }
                };
              }
            })
          }
        >
          Load More
        </Button>
      )}
    </>
  );
};

export default Launches;
```

是不是看得有点花？问题不大，其实 Apollo-Client 已经把大部分逻辑都处理好了。仔细分析：

- 首先只有在 `hasMore` 属性为 **true** 时，加载更多的按钮才会展现出来。

- `fetchMore` 函数中，我们指定了 `variables` 和 `updateQuery` ，前者是在发起本次请求时会携带的变量，即 对应`$after`，后者负责告诉客户端缓存怎么更新数据，通过将前一次数据与新的从`fetchMore`返回的结果连接在一起。

- 你可能会好奇为什么第一次没有设置变量也能成功完成请求，仔细看上面的查询语句，`$after` 这个变量没有被标明为必须提供，同时在我们的服务端中也并不要求这个参数必传。具体在 [服务端的 util.js](https://github.com/linbudu599/Apollo-Tutorial-App-ServerSide/blob/master/src/utils.js)，你可以看到如果没有传入该参数，那么就会直接返回全部结果的前 20 条。

### [fragment](https://graphql.org.cn/learn/queries-fragments.html)

`fragment` 可以用于在多个查询之间共享查询域（即返回结果）

```typescript
// 不使用fragment
const GET_LAUNCHES = gql`
  query launchList($after: String) {
    launches(after: $after) {
      cursor
      hasMore
      launches {
        id
        isBooked
        rocket {
          id
          name
        }
        mission {
          name
          missionPatch
        }
      }
    }
  }
`;

// 使用fragment
// 命名并将其定义在schema里的类型之上
const LAUNCH_TILE_DATA = gql`
  fragment LaunchTile on Launch {
    id
    isBooked
    rocket {
      id
      name
    }
    mission {
      name
      missionPatch
    }
  }
`;

const GET_LAUNCHES = gql`
  query launchList($after: String) {
    launches(after: $after) {
      cursor
      hasMore
      launches {
        ...LaunchTile
      }
    }
  }
  # import fragment
  ${LAUNCH_TILE_DATA}
`;
```

### 变更获取数据的配置

默认情况下，AC 会使用`cache-first`优先的配置，但是你可以在一些需要时刻保持最新的场景下变更为`network-only`，在 `useQuery` 中配置的哈。

```typescript
const { data, loading, error } = useQuery<GetMyTripsTypes.GetMyTrips, any>(
  GET_MY_TRIPS,
  // we want to always reflect the newest data
  { fetchPolicy: "network-only" }
);
```

### useMutation

`useQuery` 封装了查询逻辑，那么 `useMutation` 自然封装的是变更逻辑，但它不同于前者，它的第一个返回值是一个变更函数（mutate function），第二个返回值是一个元组，用于存储返回值、加载和错误状态。

以登陆为例

```typescript
export const LOGIN_USER = gql`
  mutation login($email: String!) {
    login(email: $email)
  }
`;

export default function Login() {
  const client: ApolloClient<any> = useApolloClient();
  // P为对应的语句和变量类型~
  const [login, { loading, error }] = useMutation<P></P>(LOGIN_USER, {
    onCompleted({ login }) {
      localStorage.setItem("token", login as string);
      client.writeData({ data: { isLoggedIn: true } });
    }
  });

  if (loading) return <Loading />;
  if (error) return <p>An error occurred</p>;

  return <LoginForm login={login} />;
}
```

可以看到，`useMutation` 接受了两个参数，不仅有查询语句，还有一个对象，这个对象里我们会定义 `handler`，这里的`onCompleted` 即是在加载完成后我们要进行的回调函数。

注意这里还有一个 hooks！`useApolloClient`，它的作用我觉得类似于 `useContext`，你可以通过它获取到前面在顶层组件定义的客户端实例，然后直接调用定义在上面的方法。在这里我们直接向客户端写入数据。

### 鉴权

服务端有鉴权，客户端自然也少不了。在这个例子里，我们会把`token`附加在每次请求上，当然。只需要定义一次。

```typescript
const client: ApolloClient<NormalizedCacheObject> = new ApolloClient({
  cache,
  link: new HttpLink({
    uri: "http://localhost:4000",
    headers: {
      authorization: localStorage.getItem("token")
    }
  }),
  resolvers,
  typeDefs
});
```

### 管理本地数据

通常在一个 App 中，我们不仅需要管理从后端获取的数据，还要关心本地的数据如网络加载状态，来进行相应的交互切换等等。使用 AC，我们能将本地数据也进行缓存，并可以和向 Graph Api 请求远程数据同时进行。

管理本地数据和管理远程数据类似，通过构建客户端 schema 与客户端 resolver，再通过 `@client` 指令来进行查询。

先看一个简单的：

```typescript
export const typeDefs = gql`
  extend type Query {
    isLoggedIn: Boolean!
    cartItems: [ID!]!
  }

  extend type Launch {
    isInCart: Boolean!
  }

  extend type Mutation {
    addOrRemoveFromCart(id: ID!): [ID!]!
  }
`;

type ResolverFn = (
  parent: any,
  args: any,
  { cache }: { cache: ApolloCache<any> }
) => any;

interface ResolverMap {
  [field: string]: ResolverFn;
}

interface AppResolvers extends Resolvers {
  Launch: ResolverMap;
  Mutation: ResolverMap;
}

export const resolvers: AppResolvers = {
  Launch: {
    isInCart: (launch: LaunchTileTypes.LaunchTile, _, { cache }): boolean => {
      // ...
    }
  },
  Mutation: {
    addOrRemoveFromCart: (_, { id }: { id: string }, { cache }): string[] => {
      // ...
    }
  }
};
```

- 借助 `typescript`，我们很好的规范了 resolver 的函数形状等，可以看到这里我们只定义了管理本地数据的 resolver。

- 注意这里的类型定义，我们实际上是继承了在服务端设计好的类型，通过 `extend` 关键字

- 然后回到 `index.js` ，我们需要把客户端类型定义与解析器也放进客户端里。

  ```typescript
  const client: ApolloClient<NormalizedCacheObject> = new ApolloClient({
    cache,
    link: new HttpLink({
      // 前面的服务端地址
      uri: "http://localhost:4000",
      headers: {
        authorization: localStorage.getItem("token"),
        "client-name": "Space Explorer [web]",
        "client-version": "1.0.0"
      }
    }),
    resolvers,
    typeDefs
  });
  ```

- 如何使用本地定义的 resolver？

  ```typescript
  const TOGGLE_CART = gql`
    mutation addOrRemoveFromCart($launchId: ID!) {
      addOrRemoveFromCart(id: $launchId) @client
    }
  `;
  ```

### 查询本地数据

在首屏，实际上我们就是通过查询本地数据来得知用户是否登录，并选择性的渲染出新的界面。

```typescript
const IS_LOGGED_IN = gql`
  query IsUserLoggedIn {
    isLoggedIn @client
  }
`;

function IsLoggedIn() {
  const { data } = useQuery(IS_LOGGED_IN);
  return data.isLoggedIn ? <Pages /> : <Login />;
}

injectStyles();

ReactDOM.render(
  <ApolloProvider client={client}>
    <IsLoggedIn />
  </ApolloProvider>,
  document.getElementById("root")
);
```

`@client` 会指定从客户端加载该数据。

可以看到，查询客户端数据仍然是通过 `useQuery` ，变动体现在查询语句上。

### 向服务端数据添加虚拟域

我的大概理解是，为服务端返回的数据添加虚拟域。这些虚拟域只存在客户端，并且能够使用本地数据来“装饰”服务端数据，实际上在 **管理本地数据** 一节，我们看到的 `isInCart` 即为虚拟域。举个例子：

```typescript
export const GET_LAUNCH_DETAILS = gql`
  query LaunchDetails($launchId: ID!) {
    launch(id: $launchId) {
      isInCart @client
      site
      rocket {
        type
      }
      ...LaunchTile
    }
  }
  ${LAUNCH_TILE_DATA}
`;
```

在这一查询语句中，`isInCart` 实际上不存在于服务端类型定义中，但我们可以加入这个域并指定 `@client` ，使得返回的数据中带上这一字段（实际上它来自于客户端）。

### 更新本地数据

你可以通过两种方式更新本地缓存中的数据，包括 **直接调用 client.writeData()** 方法与使用 客户端 resolver

前者通常用于写入一些简单的数据，复杂的逻辑应该交由后者处理。

在前面我们已经看到过一个在 `useMutation` 的 `onCompleted handler`里直接写入数据的例子，即

```typescript
export default function Login() {
  const client: ApolloClient<any> = useApolloClient();
  const [login, { loading, error }] = useMutation<
    LoginTypes.login,
    LoginTypes.loginVariables
  >(LOGIN_USER, {
    onCompleted({ login }) {
      localStorage.setItem("token", login as string);
      client.writeData({ data: { isLoggedIn: true } });
    }
  });

  if (loading) return <Loading />;
  if (error) return <p>An error occurred</p>;

  return <LoginForm login={login} />;
}
```

我们还可以通过 `useMutation` 的另外一个 handler 来直接写入数据，`update`，我们能通过它在一个 mutation 发生后手动更新缓存，而不用重新获取数据。

```typescript
const BookTrips: React.FC<BookTripsProps> = ({ cartItems }) => {
  const [bookTrips, { data }] = useMutation<
    BookTripsTypes.BookTrips,
    BookTripsTypes.BookTripsVariables
  >(BOOK_TRIPS, {
    variables: { launchIds: cartItems },
    refetchQueries: cartItems.map(launchId => ({
      query: GET_LAUNCH,
      variables: { launchId }
    })),

    update(cache) {
      cache.writeData({ data: { cartItems: [] } });
    }
  });

  return data && data.bookTrips && !data.bookTrips.success ? (
    <p data-testid="message">{data.bookTrips.message}</p>
  ) : (
    <Button onClick={() => bookTrips()} data-testid="book-button">
      Book All
    </Button>
  );
};
```

在这个例子里我们直接重置了 `cartItem` 的值，并且是在 `BookTrips` 这个请求发送之前。

### 总结

事实上我也觉得有点迷糊了，一路跟着写下来颇有些不求甚解的意味。但这是在使用原生的 `Apollo-Client`，希望专门的 `React-Apollo` 能给我惊喜吧

## React-Apollo

React-Apollo（简称 RA，你懂的），Apollo 客户端的首要设计原则就是兼容你正在使用的前后端工具。维护人员关注解决这些难题：GraphQL 缓存、请求管理、UI 更新。

- RA 的内部使用的是 `Redux`，所以可以很容易的集成到现有的 store，同时 redux 工具也是可用的，如 `resux-devtool` 和 `immutable.js`（dva 呢），还可以将 RA 和其他数据管理库如 `mobx` 一起使用。

- RA 和路由层是独立的，可以使用任意你喜欢的路由库（但是得适合 react）

- RA 很容易和 Next.js 结合使用，实现同构渲染 React 应用
