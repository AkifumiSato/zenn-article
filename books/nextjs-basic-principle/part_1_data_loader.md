---
title: "N+1とDataLoader"
---

## 要約

データフェッチを分割して発生するN+1フェッチは、DataLoaderで解消しましょう。

## 背景

前述の[データフェッチ コロケーション](part_1_colocation)や[並列データフェッチ](part_1_parallel_fetch)を実践し、データフェッチやコンポーネントを細かく分割していくと、ページ全体で発生するデータフェッチの管理が難しくなり2つの問題を引き起こします。

1つは重複したデータフェッチです。これについてはNext.jsの機能である[Request Memoization](https://nextjs.org/docs/app/building-your-application/caching#request-memoization)によって解消されるため、我々開発者が気にする必要はありません。

もう1つは、いわゆる**N+1なデータフェッチ**です。末端のコンポーネントでデータフェッチするよう設計すると、どうしてもN+1フェッチに繋がりがちです。以下の例では投稿の一覧を取得後、子コンポーネントで著者情報を取得しています。

```tsx
// page.tsx
import { type Post, getPosts, getUser } from "./fetcher";

export const dynamic = "force-dynamic";

export default async function Page() {
  const { posts } = await getPosts();

  return (
    <>
      <h1>N+1とDataLoader</h1>
      <h2>Posts</h2>
      <ul>
        {posts.map((post) => (
          <li key={post.id}>
            <PostItem post={post} />
          </li>
        ))}
      </ul>
    </>
  );
}

async function PostItem({ post }: { post: Post }) {
  const user = await getUser(post.userId);

  return (
    <>
      <h3>{post.title}</h3>
      <dl>
        <dt>author</dt>
        <dd>{user?.username ?? "[unknown author]"}</dd>
      </dl>
      <p>{post.body}</p>
    </>
  );
}
```

```ts
// fetcher.ts
export async function getPosts() {
  return fetch("https://dummyjson.com/posts").then(
    (res) =>
      res.json() as Promise<{
        posts: Post[];
      }>,
  );
}

type Post = {
  id: number;
  title: string;
  body: string;
  userId: number;
};

export async function getUser(id: number) {
  return fetch(`https://dummyjson.com/users/${id}`).then(
    (res) => res.json() as Promise<User>,
  );
}

type User = {
  id: number;
  username: string;
};
```

ページレンダリング時に`getPosts()`を1回と`getUser()`をN回呼び出すことになり、ページ全体では以下のようなN+1回のデータフェッチが発生します。

- `https://dummyjson.com/posts`
- `https://dummyjson.com/users/1`
- `https://dummyjson.com/users/2`
- `https://dummyjson.com/users/3`
- ...

## 設計・プラクティス

上記のようなN+1フェッチを避けるため、API側では`https://dummyjson.com/users/?id=1,2,3...`のように、idを複数指定してUser情報を一括で取得できるよう設計することが一般的です。Next.jsにおいてはこのようなエンドポイントと、[DataLoader](https://github.com/graphql/dataloader)を利用することでN+1フェッチを解消できます。

### DataLoader

DataLoaderはGraphQLサーバーなどでよく利用されるライブラリで、データアクセスをバッチ・キャッシュする機能を提供します。具体的には以下のような流れで利用します。

1. バッチ処理する関数を定義
2. DataLoaderのインスタンスを生成
3. `dataLoader.load(id)`の形で複数回呼び出すと`id`がまとめてバッチ処理に渡される
4. バッチ処理が完了すると`dataLoader.load(id)`のPromiseが解決される

以下は非常に簡単な実装例です。

```ts
async function myBatchFn(keys: readonly number[]) {
  // keysを元にデータフェッチ
}

const myLoader = new DataLoader(myBatchFn);

// 呼び出しはDataLoaderによってまとめられ、`myBatchFn([1, 2])`が呼び出される
myLoader.load(1);
myLoader.load(2);
```

### Next.jsにおけるDataLoaderの利用

Server Componentsの兄弟コンポーネントは並行レンダリングされるので、それぞれで`await myLoader.load(1);`のようにしてもDataLoaderによってバッチングされます。

DataLoaderを用いて、前述の実装例の`getUser()`を書き直してみます。

```ts
// fetcher.ts
import DataLoader from "dataloader";
import { cache } from "react";

// ...

const getUserLoader = cache(
  () => new DataLoader((keys: readonly number[]) => batchGetUser(keys)),
);

export async function getUser(id: number) {
  const userLoader = getUserLoader();
  return userLoader.load(id);
}

async function batchGetUser(keys: readonly number[]) {
  // 💡実際には`https://dummyjson.com/users`はid複数指定に未対応なので実装イメージです
  const res = await fetch(`https://dummyjson.com/users/?id=${keys.join(",")}`);
  const { users } = (await res.json()) as { users: User[] };
  return keys.map((key) => users.find((user: User) => user.id === key) ?? null);
}

// ...
```

ポイントは`getUserLoader`が`cache()`を利用していることです。DataLoaderはキャッシュ機能があるため、リクエストを跨いで共有してしまうと予期せぬデータ参照や障害につながります。そのため、実装上**リクエスト単位でDataLoaderのインスタンスを生成**する必要があります。これを実現するために、[React Cache](https://nextjs.org/docs/app/building-your-application/caching#react-cache-function)を利用しています。

上記のように実装することで、`<PostItem>`にUserデータ取得の責務を閉じたままN+1フェッチを解消することができます。

## トレードオフ

特になし
