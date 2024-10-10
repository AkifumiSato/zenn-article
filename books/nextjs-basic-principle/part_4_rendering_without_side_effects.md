---
title: "副作用のないレンダリング"
---

## 要約

Reactコンポーネントのレンダリングは副作用を含むべきではありません。Server Componentsにおいてもこれは同様で、データフェッチのみが例外として扱われます。

## 背景

Reactの最大の特徴の1つは、[宣言的](https://ja.wikipedia.org/wiki/%E5%AE%A3%E8%A8%80%E5%9E%8B%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0)であることです。従来、これは特に[宣言的UI](https://ja.react.dev/learn/reacting-to-input-with-state#how-declarative-ui-compares-to-imperative)という特徴として語られてきました。宣言的UIの前提条件として、Reactコンポーネントのレンダリングは副作用^[ここでは、他コンポーネントのレンダリングに影響しうる論理的状態の変更を指します]を持たず、純粋である必要があります。

https://ja.react.dev/learn/keeping-components-pure#side-effects-unintended-consequences

とはいえ、WebのUI実装には様々な副作用がつきものです。ReactではClient Componentsにおける副作用のハンドリングを、イベントハンドラや`useEffect()`で行うことを推奨しています。

### 並行レンダリング

React18で並行レンダリングの機能が導入されましたが、これはコンポーネントが純粋であることを前提としています。もしレンダリングに副作用が含まれるまま並行レンダリングにしてしまうと、レンダリング結果が不安定になってしまう可能性があります。

このように、Reactの多くの機能はコンポーネントが副作用を持たないことを前提としています。

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

### Cookie操作

App RouterにおいてCookie操作は典型的な副作用の1つであり、Server Componentsからは変更操作である`cookies().set()`や`cookies().delete()`は呼び出すことができません。

https://nextjs.org/docs/app/api-reference/functions/cookies

[_データ操作とServer Actions_](part_3_data_mutation)でも述べたように、Cookie操作や、APIに対するデータ変更リクエストなど変更操作はServer Actionsで行いましょう。

## トレードオフ

### レンダリングのlogging

データフェッチ同様、loggingも副作用の1つとして扱われます。loggingは他のコンポーネントのレンダリングに影響することはありませんが、Server Componentsで実装したい場合は注意が必要です。Server ComponentsはClient Componentsと違ってレンダリングされる頻度は少ないですが、Suspend（`Promise`の`throw`）によってレンダリングの中断や再開を行う可能性があり、実装次第では意図せず何度もlog出力されてしまう可能性もあります。

少々極端な例ですが、以下のコンポーネントに`promise={setTimeout(1000)}`を渡すと、2回consoleが出力されます。

```tsx
function ConcurrentComponent({ promise }: { promise: Promise<void> }) {
  console.log("render: ConcurrentComponent");
  use(promise);
  return <div>ConcurrentComponent</div>;
}
```

`use()`は`Promise`を受け取って未解決の場合`throw`し、`Promise`が解決するとそのSuspense境界を再実行します。そのため、上記実装では`conosle.log()`が2回実行されます。

このように、実装次第ではServer Componentsも再実行される可能性もあるので、loggingは`return`の直前などに実装して複数回loggingされることのないようにしましょう。

:::message
Next.js@v15RCで導入された[`after()`](https://nextjs.org/docs/app/api-reference/functions/unstable_after)は、レスポンス完了後に実行されるライフサイクルメソッドです。ただし、`after()`は実行を遅延するのみなので、上記のように`use()`より前に`after()`を実行すると、やはり2回log出力されます。
:::
