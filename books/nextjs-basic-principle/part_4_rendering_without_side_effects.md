---
title: "副作用のないレンダリング"
---

## 要約

Reactコンポーネントのレンダリングは純粋であるべきです。Server Componentsにおいてもこれは同様で、データフェッチをキャッシュして冪等にすることで純粋性を保っています。

## 背景

Reactは従来より、コンポーネントが**純粋**であることを重視してきました。Reactの最大の特徴の1つである[宣言的UI](https://ja.react.dev/learn/reacting-to-input-with-state#how-declarative-ui-compares-to-imperative)も、コンポーネントが純粋であることを前提としています。

とはいえ、WebのUI実装には様々な副作用^[ここでは、他コンポーネントのレンダリングに影響しうる論理的状態の変更を指します]がつきものです。Client Componentsでは、副作用を`useState()`や`useEffect()`などのhooksに分離することで、コンポーネントの純粋性を保てるように設計されています。

https://ja.react.dev/learn/keeping-components-pure#side-effects-unintended-consequences

### 並行レンダリング

React18で並行レンダリングの機能が導入されましたが、これはコンポーネントが純粋であることを前提としています。

https://ja.react.dev/blog/2022/03/29/react-v18#what-is-concurrent-react

もし副作用が含まれるレンダリングを並行にしてしまうと処理が不安定になりますが、副作用を含まなければレンダリングを並行にしても処理は安定します。このように、従来よりReactの多くの機能はコンポーネントが副作用を持たないことを前提としていました。

## 設計・プラクティス

React Server Componentsにおいても従来同様、**コンポーネントが純粋**であることは非常に重要です。App Routerもこの原則に沿って、各種APIが設計されています。

### データフェッチの一貫性

[_データフェッチ on Server Components_](part_1_server_components)で述べたように、App RouterにおけるデータフェッチはServer Componentsで行うことが推奨されます。データフェッチは純粋性を損なう操作の典型ですが、Server ComponentsではRequest Memoizationによってデータフェッチにおいても冪等性を実現しています。これにより、Server Componentsはデータフェッチを扱うにも関わらず、コンポーネントの純粋性を保っているように振る舞います。

Request Memoizationは拡張した`fetch()`によって実現されていますが、DBアクセスなど`fetch()`を利用しないデータフェッチについても同様に冪等性が求められます。これは`React.cache()`を利用することで簡単に実装することができます。

```ts
export const getPost = cache(async (id: number) => {
  const post = await db.query.posts.findFirst({
    where: eq(posts.id, id),
  });

  if (!post) throw new NotFoundError("Post not found");
  return post;
});
```

### Cookie操作

App RouterにおけるCookie操作も典型的な副作用の1つであり、Server Componentsからは変更操作である`cookies().set()`や`cookies().delete()`は呼び出すことができません。

https://nextjs.org/docs/app/api-reference/functions/cookies

[_データ操作とServer Actions_](part_3_data_mutation)でも述べたように、Cookie操作や、APIに対するデータ変更リクエストなど変更操作はServer Actionsで行いましょう。

## トレードオフ

### Request Memoizationのオプトアウト

Request Memoizationは`fetch()`を拡張することで実現しています。`fetch()`の拡張をやめるようなオプトアウト手段は現状ありません。ただし、Request Memoizationはメモ化なので、引数次第で都度データフェッチするようにオプトアウトすることも可能です。

```ts
// GETパラメータにランダムな値を付与する
fetch(`https://dummyjson.com/todos/random?_hash=${Math.random()}`);
// 毎回異なるAbortSignalインスタンスを指定する
const controller = new AbortController();
fetch(`https://dummyjson.com/todos/random`, { signal: controller.signal });
```
