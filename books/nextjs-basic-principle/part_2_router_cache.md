---
title: "Router Cache"
---

## 要約

アプリケーション特性に応じてRouter Cacheの`staleTimes`を適切に設定しつつ、適宜必要なタイミングでrevalidateしましょう。

## 背景

[Router Cache](https://nextjs.org/docs/app/building-your-application/caching#router-cache)は、App Routerにおけるクライアントサイドキャッシュで[RSC Payload](https://nextjs.org/docs/app/building-your-application/rendering/server-components#how-are-server-components-rendered)、つまりServer Componentsのレンダリング結果を保持しています。Router Cacheはprefetchやsoft navigation時に更新され、有効期間内であれば再利用されます。

v14.1以前のApp RouterではRouter Cacheの有効期間を開発者から設定することはできず、`<Link>`の`prefetch`指定などに基づいてキャッシュの有効期限は**30秒か5分**とされていました。これに対しNext.jsリポジトリの[Discussion](https://github.com/vercel/next.js/discussions/54075)上では、Router Cacheをアプリケーション毎に適切に設定できるようにして欲しいという要望が相次いでいました。

## 設計・プラクティス

Next.jsの[v14.2](https://nextjs.org/blog/next-14-2#caching-improvements)にて、Router Cacheの有効期間を設定する[`staleTimes`](https://nextjs.org/docs/app/api-reference/next-config-js/staleTimes)が実験的機能として導入されました。これにより、開発者はアプリケーション特性に応じてRouter Cacheの有効期間を適切に設定することができるようになりました。

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    staleTimes: {
      dynamic: 0, // default: 30
      static: 180, // default: 300
    },
  },
};

export default nextConfig;
```

Router CacheはApp Routerの中でも混乱が多く見られる部分なので、この設定については**必ず設定を検討すること**を筆者はお勧めします。検討した上で初期値のままにすることも良いと思います。重要なのはアプリケーション特性に応じた選択と、現在の設定値に対する認識です。

### `staleTimes`の設定

`staleTimes`の設定は[ドキュメント](https://nextjs.org/docs/app/api-reference/next-config-js/staleTimes)によると以下のように対応していることになります。

| 項目    | `<Link prefetch=?>` | デフォルト |
| ------- | ------------------- | ---------- |
| dynamic | `undefined`         | 30s        |
| static  | `true`              | 5m         |

:::message
[トレードオフ](#トレードオフ)にて後述しますが、実際にはRouter Cacheの動作は非常に複雑なためこの対応の限りではありません。
:::

多くの場合、変更を考えるべくは`dynamic`の方になります。クライアントサイドで動的なコンテンツをキャッシュ保持する際にどこまで許容できるかという観点で設定を考えましょう。

### 任意のタイミングでrevalidate

`staleTimes`以外でRouter Cacheを任意に破棄するには、以下3つの方法があります。

- [`router.refresh()`](https://nextjs.org/docs/app/building-your-application/caching#routerrefresh)
- Server Actionsで`revalidatePath()`/`revalidateTag()`
- Server Actionsで`cookies.set()`/`cookies.delete()`

Router Cacheを任意のタイミングで破棄したい多くのユースケースはユーザーによるデータ操作時です。データ操作を行うServer Actions内で`revalidatePath()`や`cookies.set()`を呼び出しているなら特に追加で実装する必要はありません。一方これらを呼び出していない場合には、データ操作のsubmit完了後にClient Components側で`router.refresh()`を呼び出すなどの対応を行いましょう。

特に`revalidatePath()`/`revalidateTag()`はサーバー側キャッシュだけでなくRouter Cacheにも影響を及ぼすことは直感的ではないので、よく覚えておきましょう。

## トレードオフ

### ドキュメントにはないRouter Cacheの挙動

Router Cacheの挙動はドキュメントにない挙動をすることも多く、非常に複雑です。特に筆者が注意しておくべき点として認識してるものを以下にあげます。

- ブラウザバック時は`staleTimes`の値にかかわらず、必ずRouter Cacheが利用される(キャッシュ破棄後であれば再取得する)
- `staleTimes.dynamic`の基準時間は「キャッシュが保存された時間」か「キャッシュが最後に利用された時間」
- `staleTimes.static`の基準時間は「キャッシュが保存された時間」のみ

より詳細な挙動を知りたい方は、筆者の[過去の記事](https://zenn.dev/akfm/articles/next-app-router-client-cache)をご参照ください。
