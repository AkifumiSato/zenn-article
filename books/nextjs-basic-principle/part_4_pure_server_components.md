---
title: "Server Componentsの純粋性"
---

## 要約

Reactコンポーネントのレンダリングは純粋であるべきです。Server Componentsにおいてもこれは同様で、データフェッチをメモ化することで純粋性を保つことができます。

## 背景

Reactは従来より、コンポーネントが**純粋**であることを重視してきました。Reactの最大の特徴の1つである[宣言的UI↗︎](https://ja.react.dev/learn/reacting-to-input-with-state#how-declarative-ui-compares-to-imperative)も、コンポーネントが純粋であることを前提としています。

とはいえ、WebのUI実装には様々な副作用^[ここでは、他コンポーネントのレンダリングに影響しうる論理的状態の変更を指します]がつきものです。Client Componentsでは、副作用を`useState()`や`useEffect()`などのhooksに分離することで、コンポーネントの純粋性を保てるように設計されています。

https://ja.react.dev/learn/keeping-components-pure#side-effects-unintended-consequences

### 並行レンダリング

React18で並行レンダリングの機能が導入されましたが、これはコンポーネントが純粋であることを前提としています。

https://ja.react.dev/blog/2022/03/29/react-v18#what-is-concurrent-react

もし副作用が含まれるレンダリングを並行にしてしまうと、処理結果が不安定になりますが、副作用を含まなければレンダリングを並行にしても結果は安定します。このように、従来よりReactの多くの機能は、コンポーネントが副作用を持たないことを前提としていました。

## 設計・プラクティス

RSCにおいても従来同様、**コンポーネントが純粋**であることは非常に重要です。Next.jsもこの原則に沿って、各種APIが設計されています。

### データフェッチの一貫性

[データフェッチ on Server Components](part_1_server_components)で述べたように、Next.jsにおけるデータフェッチはServer Componentsで行うことが推奨されます。本来、データフェッチは純粋性を損なう操作の典型です。

```tsx
async function getRandomTodo() {
  // リクエストごとにランダムなTodoを返すAPI
  const res = await fetch("https://dummyjson.com/todos/random");
  return (await res.json()) as Todo;
}
```

上記の`getRandomTodo()`は呼び出しごとに異なるTodoを返す可能性が高く、そもそもリクエストに失敗する可能性もあるため戻り値は不安定です。このようなデータフェッチを内部的に呼び出す関数は、同じ入力（引数）でも出力（戻り値）が異なる可能性があり、純粋ではありません。

Next.jsは[Request Memoization↗︎](https://nextjs.org/docs/app/guides/caching#request-memoization)によって入力に対する出力を冪等に保ち、データフェッチをサポートしながらもレンダリングの範囲ではコンポーネントの純粋性を保てるよう設計されています。

```tsx
export async function ComponentA() {
  const todo = await getRandomTodo();
  return <>...</>;
}

export async function ComponentB() {
  const todo = await getRandomTodo();
  return <>...</>;
}

// ...

export default function Page() {
  // ランダムなTodoは1度だけ取得され、
  // 結果<ComponentA>と<ComponentB>は同じTodoを表示する
  return (
    <>
      <ComponentA />
      <ComponentB />
    </>
  );
}
```

### `cache()`によるメモ化

Request Memoizationは`fetch()`を拡張することで実現されていますが、DBアクセスなど`fetch()`を利用しないデータフェッチについても同様に純粋性は重要です。これは`React.cache()`を利用することで簡単に実装することができます。

```ts
export const getPost = cache(async (id: number) => {
  const post = await db.query.posts.findFirst({
    where: eq(posts.id, id),
  });

  if (!post) throw new NotFoundError("Post not found");
  return post;
});
```

ページを通して1回だけ発生するデータフェッチに対してなら、上記のように`cache()`でメモ化する必要はないように感じられるかもしれませんが、筆者は基本的にメモ化しておくことを推奨します。あえてメモ化したくない場合には、後々の改修で複数回呼び出すことになった際に一貫性が破綻するリスクが伴います。

### Cookie操作

Next.jsにおけるCookie操作も典型的な副作用の1つであり、Server Componentsからは変更操作であるCookieの`.set()`や`.delete()`は呼び出すことができません。

https://nextjs.org/docs/app/api-reference/functions/cookies

[データ操作とServer Actions](part_3_data_mutation)でも述べたように、Cookie操作や、APIに対するデータ変更リクエストなど変更操作はServer Actionsで行いましょう。

## トレードオフ

### Request Memoizationのオプトアウト

Request Memoizationは`fetch()`を拡張することで実現しています。`fetch()`の拡張をやめるようなオプトアウト手段は現状ありません。ただし、`fetch()`に渡す引数次第でRequest Memoizationをオプトアプトして都度データフェッチすることは可能です。

```ts
// クエリー文字列にランダムな値を付与する
fetch(`https://dummyjson.com/todos/random?_hash=${Math.random()}`);
// `signal`を指定する
const controller = new AbortController();
const signal = controller.signal;
fetch(`https://dummyjson.com/todos/random`, { signal });
```
