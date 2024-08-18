---
title: "並行データフェッチ"
---

## 要約

以下のパターンを駆使して、データフェッチが可能な限り並行になるよう設計しましょう。

- [データフェッチ単位のコンポーネント分割](#データフェッチ単位のコンポーネント分割)
- [並行`fetch()`](#並行fetch)
- [preloadパターン](#preloadパターン)

## 背景

「商品情報を取得してからじゃないと出品会社情報が取得できない」と言ったように、データフェッチ間に依存関係がある場合、データフェッチ処理自体は直列(ウォーターフォール)に実行せざるを得ません。

一方データ間に依存関係がない場合、当然ながらデータフェッチを並行化した方が優れたパフォーマンスを得られます。以下は[公式ドキュメント](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#parallel-and-sequential-data-fetching)にあるデータフェッチの並行化による速度改善のイメージ図です。

![water fall data fetch](/images/nextjs-basic-principle/sequential-fetching.png)

## 設計・プラクティス

App Routerにおけるデータフェッチの並行化にはいくつかの実装パターンがあります。コードの凝集度を考えると、まずは可能な限り[データフェッチ単位のコンポーネント分割](#データフェッチ単位のコンポーネント分割)を行うことがベストです。ただし、必ずしもコンポーネントが分割可能とは限らないので他のパターンについてもしっかり理解しておきましょう。

### データフェッチ単位のコンポーネント分割

データ間に依存関係がなく参照単位も異なる場合には、データフェッチを行うコンポーネント自体分割し、兄弟コンポーネントとすることを検討しましょう。このような非同期な兄弟コンポーネントは、並行にレンダリングされます。

```tsx
function Page({ params: { id } }: { params: { id: string } }) {
  return (
    <>
      <PostBody postId={id} />
      <Comments postId={id} />
    </>
  );
}

async function PostBody({ postId }: { postId: string }) {
  const post = await fetch(`https://dummyjson.com/posts/${postId}`).then(
    (res) => res.json(),
  );
  // ...
}

async function Comments({ postId }: { postId: string }) {
  const post = await fetch(
    `https://dummyjson.com/posts/${postId}/comments`,
  ).then((res) => res.json());
  // ...
}
```

上記の実装例では`<PostBody />`と`<Comments />`は兄弟要素のため、並行レンダリングされデータフェッチも並行となります。

### 並行`fetch()`

データフェッチ順には依存関係がなくとも参照の単位が不可分な場合には、`Promise.all()`(もしくは`Promise.allSettled()`)と`fetch()`を組み合わせることで、複数のデータフェッチを並行に実行できます。

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

コンポーネント構造上兄弟関係ではなく親子関係にせざるを得ない場合も、データフェッチにウォーターフォールが発生します。このようなウォーターフォールは、Request Memoizationを活用した[preloadパターン](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#preloading-data)を利用することで、コンポーネントの親子関係を超えて並行データフェッチが実現できます。

サーバー間通信は物理的距離・潤沢なネットワーク環境などの理由から安定して高速な傾向にあり、ウォーターフォールがパフォーマンスに及ぼす影響はクライアントサイドと比較すると小さくなる傾向にあります。それでもパフォーマンス計測した時に無視できない遅延を含む場合などには、このpreloadパターンが有用です。

```ts :app/fetcher.ts
import "server-only";

export const preloadCurrentUser = () => {
  void getCurrentUser();
};

export async function getCurrentUser() {
  const res = await fetch("https://dummyjson.com/user/me");
  return res.json() as User;
}
```

```tsx :app/products/[id]/page.tsx
export default function Page({ params: { id } }: { params: { id: string } }) {
  // `<Product>`や`<Comments>`のさらに子孫で`user`を利用するため、親コンポーネントでpreloadする
  preloadCurrentUser();

  return (
    <>
      <Product productId={id} />
      <Comments productId={id} />
    </>
  );
}
```

上記実装例では`<Product>`や`<Comments>`の子孫でUser情報を利用するため、ページレベルで`preloadCurrentUser()`することで、`<Product>`と`<Comments>`のレンダリングと並行してUser情報のデータフェッチが実行されます。

ただし、preloadパターンを利用した後で`<Product>`や`<Comments>`からUser情報が参照されなくなった場合、`preloadCurrentUser()`が残っていると不要なデータフェッチが発生します。このパターンを利用する際には、無駄なpreloadが残ってしまうことのないよう注意しましょう。

## トレードオフ

### N+1データフェッチ

データフェッチ単位を小さくコンポーネントに分割していくと**N+1データフェッチ**が発生する可能性があります。この点については次の章の[_N+1とDataLoader_](part_1_data_loader)で詳しく解説します。
