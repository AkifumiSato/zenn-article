---
title: "並列データフェッチ"
---

## 要約

データフェッチが可能な限り並列になるよう設計しましょう。

## 背景

データフェッチ間に依存関係がある場合、データフェッチ処理自体は直列(ウォーターフォール)に実行せざるを得ません。一方データ間に依存関係がない場合、当然ながらデータフェッチを並列化した方が優れたパフォーマンスを得られます。以下は[公式ドキュメント](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#parallel-and-sequential-data-fetching)にあるデータフェッチの並列化による速度改善のイメージ図です。

![water fall data fetch](/images/nextjs-basic-principle/sequential-fetching.png)

## 設計・プラクティス

App Routerにおけるデータフェッチの並列化にはいくつかの実装パターンがあります。コードの凝集度を考えると、まずは可能な限り**データフェッチ単位のコンポーネント分割**を行うことがベストです。ただし、必ずしもコンポーネントが分割可能とは限らないので他のパターンについてもしっかり理解しておきましょう。

:::message
以降の解説においては、「並行」と「並列」の違いに注意してください。
Reactは並行レンダリングをサポートしていますが、レンダリング処理自体はメインスレッドで行われるため並列処理はできません。一方ブラウザやNode.jsを介して行われるhttp通信は並列実行が可能です。
詳しくは[こちらの記事](https://zenn.dev/estra/books/js-async-promise-chain-event-loop/viewer/17-epasync-static-method)をご参照ください。
:::

### データフェッチ単位のコンポーネント分割パターン

データ間に依存関係がなく、参照単位も異なる場合にはデータフェッチを行うコンポーネント自体分割し、兄弟コンポーネントとすることを検討しましょう。React Server Componentsにおいて、非同期な兄弟コンポーネントは並行にレンダリングされます。

以下は`<PostBody />`と`<Comments />`が並行レンダリングされ、データフェッチが並列に実行される例です。

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

### `Promise.all()`+`fetch()`パターン

それぞれのデータを組み合わせて加工する必要がある場合や、参照の単位が不可分な場合には`Promise.all()`と`fetch()`を組み合わせることで、複数のデータフェッチを並列に実行できます。

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

コンポーネント分割時に兄弟関係ではなく親子関係にせざるを得ない場合、データフェッチにウォーターフォールが発生します。[Request Memoization](part_1_request_memoization)で述べたpreloadパターンを利用すれば、コンポーネントの親子関係を超えて並列データフェッチが実現できます。

```tsx
import "server-only";

export const preload = (id: string) => {
  void getProduct(id);
};

export async function getProduct(id: string) {
  const res = await fetch(`https://dummyjson.com/products/${id}`);
  return res.json();
}

// 親コンポーネント
export async function ParentComponent({ id }: { id: string }) {
  preload(id);

  // ...
}
```

## トレードオフ

データフェッチ単位を小さくコンポーネントに分割していくと**N+1フェッチ**が発生しがちになります。この点については次の[N+1とDataLoader](part_1_data_loader)で詳しく解説します。