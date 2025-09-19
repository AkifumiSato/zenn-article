---
title: "Router Cache"
---

## 要約

Router Cacheはクライアントサイドのキャッシュで、prefetch時やSoft Navigation時に更新されます。アプリケーション特性に応じてRouter Cacheの有効期間である`staleTimes`を適切に設定しつつ、適宜必要なタイミングでrevalidateしましょう。

## 背景

[Router Cache↗︎](https://nextjs.org/docs/app/guides/caching#client-side-router-cache)は、App Routerにおけるクライアントサイドキャッシュで、Server Componentsのレンダリング結果である[RSC Payload↗︎](https://nextjs.org/docs/app/getting-started/server-and-client-components#on-the-server)を保持しています。Router Cacheはprefetchやsoft navigation時に更新され、有効期間内であれば再利用されます。

v14.1以前のApp RouterではRouter Cacheの有効期間を開発者から設定することはできず、`<Link>`の`prefetch`指定などに基づいてキャッシュの有効期限は**30秒か5分**とされていました。これに対しNext.jsリポジトリの[Discussion↗︎](https://github.com/vercel/next.js/discussions/54075)上では、Router Cacheをアプリケーション毎に適切に設定できるようにして欲しいという要望が相次いでいました。

## 設計・プラクティス

Next.jsの[v14.2↗︎](https://nextjs.org/blog/next-14-2#caching-improvements)にて、Router Cacheの有効期間を設定する`staleTimes`がexperimentalで導入されました。これにより、開発者はアプリケーション特性に応じてRouter Cacheの有効期間を適切に設定することができるようになりました。

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    staleTimes: {
      dynamic: 10,
      static: 180,
    },
  },
};

export default nextConfig;
```

### `staleTimes`の設定

`staleTimes`の設定は[ドキュメント↗︎](https://nextjs.org/docs/app/api-reference/config/next-config-js/staleTimes)によると以下のように対応していることになります。

| 項目    | `<Link prefetch=?>` | デフォルト(v14) | デフォルト(v15) |
| ------- | ------------------- | --------------- | --------------- |
| dynamic | `undefined`         | 30s             | 0s              |
| static  | `true`              | 5m              | 5m              |

:::message
[#トレードオフ](#トレードオフ)にて後述しますが、実際にはRouter Cacheの動作は非常に複雑なためこの対応の限りではありません。
:::

多くの場合、変更を考えるべくは`dynamic`の方になります。v14では特に、デフォルトだと30sとなっているために多くの混乱が見られました。利用するNext.jsのバージョンごとの挙動に注意して、キャッシュの有効期限としてどこまで許容できるか考えて適切に設定しましょう。

### 任意のタイミングでrevalidate

`staleTimes`以外でRouter Cacheを任意に破棄するには、以下3つの方法があります。

- [`router.refresh()`↗︎](https://nextjs.org/docs/app/guides/caching#routerrefresh)
- Server Actions^[データ操作を伴うServer Functionsは、**Server Actions**と呼ばれます。[参考↗︎](https://nextjs.org/docs/app/getting-started/updating-data#what-are-server-functions)]で`revalidatePath()`/`revalidateTag()`
- Server Actionsで`cookies.set()`/`cookies.delete()`

Router Cacheを任意のタイミングで破棄したい多くのユースケースは、ユーザーによるデータ操作時です。Server Actions内で`revalidatePath()`や`cookies.set()`を呼び出しているなら特に追加実装する必要はありません。一方これらを呼び出していない場合には、データ操作のsubmit完了後にClient Components側で`router.refresh()`を呼び出すなどの対応を行いましょう。

特に`revalidatePath()`/`revalidateTag()`はサーバー側キャッシュだけでなくRouter Cacheにも影響を及ぼすことは直感的ではないので、よく覚えておきましょう。

:::message
Router Cacheはクライアントサイドのキャッシュのため、上記によるキャッシュ破棄はあくまでユーザー端末内で起こります。**全ユーザーのRouter Cacheを同時に破棄する方法はありません**。

例えばあるユーザーがServer Actions経由で`revalidatePath()`を呼び出しても、他のユーザーはそれぞれ`staleTimes.static`(デフォルト: 5分)または`staleTimes.dynamic`(デフォルト: 0秒)経過後の画面更新までRouter Cacheを参照し続けます。
:::

## トレードオフ

### ドキュメントにはないRouter Cacheの挙動

Router Cacheの挙動はドキュメントにない挙動をすることも多く、非常に複雑です。特に筆者が注意しておくべき点として認識してるものを以下にあげます。

- ブラウザバック時は`staleTimes`の値にかかわらず、必ずRouter Cacheが利用される(キャッシュ破棄後であれば再取得する)
- `staleTimes.dynamic`に指定する値は、「キャッシュが保存された時刻」か「キャッシュが最後に利用された時刻」からの経過時間
- `staleTimes.static`に指定する値は、「キャッシュが保存された時刻」からの経過時間のみ

より詳細な挙動を知りたい方は、少々古いですが筆者の過去記事をご参照ください。

https://zenn.dev/akfm/articles/next-app-router-client-cache
