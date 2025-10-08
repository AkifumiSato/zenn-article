---
title: "N+1とDataLoader"
---

## 要約

コンポーネント単位の独立性を高めるとN+1データフェッチが発生しやすくなるので、DataLoaderのバッチ処理を利用して解消しましょう。

:::message
本章の内容は、バックエンドAPI側でもN+1データフェッチに対処するためのエンドポイントが実装されていることを前提としています。
:::

## 背景

前述の[データフェッチ コロケーション](part_1_colocation)や[並行データフェッチ](part_1_concurrent_fetch)を実践し、データフェッチやコンポーネントを細かく分割していくと、ページ全体で発生するデータフェッチの管理が難しくなり2つの問題を引き起こします。

1つは重複したデータフェッチです。これについてはNext.jsの機能である[Request Memoization](part_1_request_memoization)によって解消されるため、前述のようにデータフェッチ層を分離・共通化していればほとんど問題ありません。

もう1つは、いわゆる**N+1**^[参考: [[解説] SQLクエリのN+1問題↗︎](https://qiita.com/muroya2355/items/d4eecbe722a8ddb2568b)]なデータフェッチです。データフェッチを細粒度に分解していくと、N+1データフェッチになる可能性が高まります。

以下の例では投稿の一覧を取得後、子コンポーネントで著者情報を取得しています。

```tsx :page.tsx
import { type Post, getPosts, getUser } from "./fetcher";

export const dynamic = "force-dynamic";

export default async function Page() {
  const { posts } = await getPosts();

  return (
    <>
      <h1>Posts</h1>
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

```ts :fetcher.ts
export async function getPosts() {
  const res = await fetch("https://dummyjson.com/posts");
  return (await res.json()) as {
    posts: Post[];
  };
}

type Post = {
  id: number;
  title: string;
  body: string;
  userId: number;
};

export async function getUser(id: number) {
  const res = await fetch(`https://dummyjson.com/users/${id}`);
  return (await res.json()) as User;
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

上記のようなN+1データフェッチを避けるため、API側では`https://dummyjson.com/users/?id=1&id=2&id=3...`のように、idを複数指定してUser情報を一括で取得できるよう設計するパターンがよく知られています。

このようなバックエンドAPIと、Next.js側で[DataLoader↗︎](https://github.com/graphql/dataloader)を利用することで前述のようなN+1データフェッチを解消することができます。

### DataLoader

DataLoaderはGraphQLサーバーなどでよく利用されるライブラリで、データアクセスをバッチ処理・キャッシュする機能を提供します。具体的には以下のような流れで利用します。

1. バッチ処理する関数を定義
2. DataLoaderのインスタンスを生成
3. 短期間^[バッチングスケジュールの詳細は[公式の説明↗︎](https://github.com/graphql/dataloader?tab=readme-ov-file#batch-scheduling)を参照ください。]に`dataLoader.load(id)`を複数回呼び出すと、`id`の配列がバッチ処理に渡される
4. バッチ処理が完了すると`dataLoader.load(id)`のPromiseが解決される

以下は非常に簡単な実装例です。

```ts
async function myBatchFn(keys: readonly number[]) {
  // keysを元にデータフェッチ
  // 実際にはdummyjsonはid複数指定に未対応なのでイメージです
  const res = await fetch(
    `https://dummyjson.com/posts/?${keys.map((key) => `id=${key}`).join("&")}`,
  );
  const { posts } = (await res.json()) as { posts: Post[] };
  return posts;
}

const myLoader = new DataLoader(myBatchFn);

// 呼び出しはDataLoaderによってまとめられ、`myBatchFn([1, 2, 3])`が呼び出される
const posts = await Promise.all([
  myLoader.load(1),
  myLoader.load(2),
  myLoader.load(3),
]);
```

### Next.jsにおけるDataLoaderの利用

Server Componentsの兄弟もしくは兄弟の子孫コンポーネントは並行レンダリングされるので、それぞれで`await myLoader.load(1);`のようにしてもDataLoaderによってバッチングされます。

DataLoaderを用いて、前述の実装例の`getUser()`を書き直してみます。

```ts :fetcher.ts
import DataLoader from "dataloader";
import * as React from "react";

// ...

const getUserLoader = React.cache(
  () => new DataLoader((keys: readonly number[]) => batchGetUser(keys)),
);

export async function getUser(id: number) {
  const userLoader = getUserLoader();
  return userLoader.load(id);
}

async function batchGetUser(keys: readonly number[]) {
  // keysを元にデータフェッチ
  // 実際にはdummyjsonはid複数指定に未対応なのでイメージです
  const res = await fetch(
    `https://dummyjson.com/users/?${keys.map((key) => `id=${key}`).join("&")}`,
  );
  const { users } = (await res.json()) as { users: User[] };
  return users;
}

// ...
```

ポイントは`getUserLoader`が`React.cache()`を利用していることです。DataLoaderはキャッシュ機能があるため、ユーザーからのリクエストを跨いでインスタンスを共有してしまうと予期せぬデータ共有につながります。そのため、**ユーザーからのリクエスト単位でDataLoaderのインスタンスを生成**する必要があり、これを実現するために[React Cache↗︎](https://ja.react.dev/reference/react/cache)を利用しています。

:::message alert
Next.jsやReactのコアメンテナである[Sebastian Markbåge氏↗︎](https://bsky.app/profile/sebmarkbage.calyptus.eu)の過去ツイート（アカウントごと削除済み）によると、React Cacheがリクエスト単位で保持される仕様は将来的に保障されたものではないようです。執筆時点では他に情報がないため、上記実装を参考にする場合には必ず動作確認を行ってください。
:::

上記のように実装することで、`getUser()`のインターフェースは変えずにN+1データフェッチを解消することができます。

## トレードオフ

### Eager Loadingパターン

ここで紹介した設計パターンはいわゆる**Lazy Loading**パターンの一種です。バックエンドAPI側の実装・パフォーマンス観点からLazy Loadingが適さない場合は**Eager Loading**パターン、つまりN+1の最初の1回のリクエストで関連する必要な情報を全て取得することを検討しましょう。

Eager Loadingパターンは偏りすぎると、密結合で責務が大きすぎるいわゆる**God API**になってしまいます。これらの詳細については次章の[細粒度のREST API設計](part_1_fine_grained_api_design)で解説します。
