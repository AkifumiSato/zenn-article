---
title: "副作用のないレンダリング"
---

## 要約

Reactコンポーネントのレンダリングは副作用を含むべきでありません。Server Componentsにおいてもこれは同様で、データフェッチのみが例外として扱われます。

## 背景

Reactの最大の特徴の1つは、[宣言的](https://ja.wikipedia.org/wiki/%E5%AE%A3%E8%A8%80%E5%9E%8B%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0)であることです。従来、これは特に[宣言的UI](https://ja.react.dev/learn/reacting-to-input-with-state#how-declarative-ui-compares-to-imperative)という特徴として語られてきました。宣言的UIの前提条件として、Reactコンポーネントのレンダリングは副作用^[ここでは、他コンポーネントのレンダリングに影響しうる論理的状態の変更を指します]を持たず、純粋である必要があります。

https://ja.react.dev/learn/keeping-components-pure#side-effects-unintended-consequences

### Client Componentsにおける副作用

とはいえ、WebのUI実装には様々な副作用がつきものです。Reactではこのような副作用を扱う場所として、イベントハンドラや`useEffect()`を推奨しています。

https://ja.react.dev/learn/keeping-components-pure#where-you-\_can\_-cause-side-effects

## 設計・プラクティス

Server Componentsも同様にReactコンポーネントであり、**データフェッチを除く副作用**を含むべきではありません。App Routerもこの原則に沿って、各種APIが設計されています。

### データフェッチの負荷軽減と一貫性

[_データフェッチ on Server Components_](part_1_server_components)で述べたように、App RouterにおけるデータフェッチはServer Componentsで行うことが推奨されます。外部通信の処理は副作用の1つと考えられることが多いですが、Server Componentsではデータフェッチに限り扱うことができます。その他の副作用はServer Actionsを通じて行うことが推奨されます。

また、App RouterではRequest Memoizationと呼ばれる`fetch()`のメモ化を提供しており、これによりデータアクセス負荷の軽減とレンダリングにおける一貫性を実現しています。これは`React.cache()`を利用することで実現されており、DBアクセスなどでもこれを利用して実装することが推奨されます。

```ts
export const getPost = cache(async (id: number) => {
  const post = await db.query.posts.findFirst({
    where: eq(posts.id, id),
  });

  if (!post) throw new NotFoundError("Post not found");
  return post;
});
```

### 並行レンダリング

TBW

### Cookie操作

TBW

## トレードオフ

### リクエストごとのロギング

TBW
