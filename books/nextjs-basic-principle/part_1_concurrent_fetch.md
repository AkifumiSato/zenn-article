---
title: "並行データフェッチ"
---

## 要約

以下のパターンを駆使して、データフェッチが可能な限り並行になるよう設計しましょう。

- [データフェッチ単位のコンポーネント分割](#データフェッチ単位のコンポーネント分割)
- [並行`fetch()`](#並行fetch)
- [preloadパターン](#preloadパターン)

## 背景

データフェッチ由来のデータ間に依存関係がある場合、データフェッチは直列(ウォーターフォール)に実行せざるを得ません。

一方、データ間に依存関係がない場合、データフェッチを並行化すれば優れたパフォーマンスを得られます。以下は[公式ドキュメント↗︎](https://nextjs.org/docs/14/app/building-your-application/data-fetching/patterns#parallel-and-sequential-data-fetching)にあるデータフェッチの並行化による速度改善のイメージ図です。

![water fall data fetch](/images/nextjs-basic-principle/sequential-fetching.png)

## 設計・プラクティス

Next.jsにおけるデータフェッチの並行化にはいくつかの実装パターンがあります。コードの凝集度を考えると、まずは可能な限り**データフェッチ単位のコンポーネント分割**を行うことがベストです。ただし、必ずしもコンポーネントが分割可能とは限らないので他のパターンについてもしっかり理解しておきましょう。

### データフェッチ単位のコンポーネント分割

データ間に依存関係がなく参照単位も異なる場合には、データフェッチを行うコンポーネント自体分割することを検討しましょう。

非同期コンポーネントは兄弟もしくは兄弟の子孫コンポーネントとして配置されてる場合、並行にレンダリングされます^[非同期コンポーネントがネストしている場合はコンポーネントの実行が直列になります。]。

```tsx
function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;

  return (
    <>
      <PostBody postId={id} />
      <CommentsWrapper>
        <Comments postId={id} />
      </CommentsWrapper>
    </>
  );
}

async function PostBody({ postId }: { postId: string }) {
  const res = await fetch(`https://dummyjson.com/posts/${postId}`);
  const post = (await res.json()) as Post;
  // ...
}

async function Comments({ postId }: { postId: string }) {
  const res = await fetch(`https://dummyjson.com/posts/${postId}/comments`);
  const comments = (await res.json()) as Comment[];
  // ...
}
```

上記の実装例では`<PostBody />`と`<Comments />`およびその子孫は並行レンダリングされるので、データフェッチも並行となります。

### 並行`fetch()`

データフェッチ順には依存関係がなくとも参照の単位が不可分な場合には、`Promise.all()`や`Promise.allSettled()`を利用することで、複数のデータフェッチを並行に実行できます。

```tsx
async function Page() {
  const [user, posts] = await Promise.all([
    fetch(`https://dummyjson.com/users/${id}`).then((res) => res.json()),
    fetch(`https://dummyjson.com/posts/users/${id}`).then((res) => res.json()),
  ]);

  // ...
}
```

### preloadパターン

コンポーネント構造上親子関係にせざるを得ない場合も、データフェッチにウォーターフォールが発生します。このようなウォーターフォールは、Request Memoizationを活用した[preloadパターン↗︎](https://nextjs.org/docs/app/getting-started/fetching-data#preloading-data)を利用することで、並行データフェッチを実現できます。

:::message
サーバー間通信は物理的距離・潤沢なネットワーク環境などの理由から安定して高速な傾向にあり、ウォーターフォールがパフォーマンスに及ぼす影響はクライアントサイドと比較すると小さくなる傾向にあります。それでも無視できないパフォーマンスボトルネックがある場合には、このpreloadパターンが有用です。
:::

```ts :app/fetcher.ts
import "server-only";

export const preloadCurrentUser = () => {
  // preloadなので`await`しない
  void getCurrentUser();
};

export async function getCurrentUser() {
  const res = await fetch("https://dummyjson.com/user/me");
  return res.json() as User;
}
```

```tsx :app/products/[id]/page.tsx
export default function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;

  // `<Product>`や`<Comments>`のさらに子孫で`user`を利用するため、
  // 親コンポーネントでpreloadする
  preloadCurrentUser();

  return (
    <>
      <Product productId={id} />
      <Comments productId={id} />
    </>
  );
}
```

上記実装例では可読性のためにpreload専用の関数として`preloadCurrentUser()`を定義しています。ページレベルで`preloadCurrentUser()`することで、`<Product>`と`<Comments>`のレンダリングと並行してUser情報のデータフェッチが実行されます。

ただし、preloadパターンを利用した後で`<Product>`や`<Comments>`からUser情報が参照されなくなった場合、`preloadCurrentUser()`が残っていると不要なデータフェッチが発生します。このパターンを利用する際には、無駄なpreloadが残ってしまうことのないよう注意しましょう。

## トレードオフ

### N+1データフェッチ

データフェッチ単位を小さくコンポーネントに分割していくと**N+1データフェッチ**が発生する可能性があります。この点については次の章の[N+1とDataLoader](part_1_data_loader)で詳しく解説します。
