---
title: "Router Cache"
---

## 要約

アプリケーション特性に応じてRouter Cacheの`staleTimes`を適切に設定し、必要に応じてrevalidateしましょう。

## 背景

- Router CacheはApp Routerにおけるクライアントサイドキャッシュで、RSC Payloadを保持しています。
- 従来App RouterではRouter Cacheの有効期間を設定することができず、いくつかの条件に基づいて30sか5mの期限が設定されていました。
- これに対しNext.jsのDiscussionでは、Router Cacheをアプリケーション毎に適切に設定できるよう要望が相次ぎました。

## 設計・プラクティス

- v14時点ではまだexperimentalですが、Router Cacheの有効期間を設定する`staleTimes`が導入されました。
- 開発者はアプリケーション特性に応じてこれを適切に設定する必要があります。
- Router CacheはApp Routerの中でも混乱が多く見られる部分なので、開発者はこの設定についてどうすべきか必ず考えることをお勧めします。
- `staleTimes`とは別に任意のタイミングでRouter Cacheを破棄したい時には、`router.refresh()`やオンデマンドrevalidateで対応できます。

### `staleTimes`の設定

- `staleTimes`には`static`と`dynamic`の2つの設定があります。
- これはstatic/dynamic rendering**ではなく**、`<Link>`の`prefetch` propsによって決定されるlivenessと呼ばれるものです。

:::message
厳密にはRouter Cacheの動作は上記static/dynamicの2つで説明できるものではなく、キャッシュの参照が初回かどうか・dynamic renderingかどうか・ブラウザバックかどうかなど様々な条件に基づいて詳細な動作が決定されるため、Router Cacheの仕様は非常に複雑です。
より詳細にこれらの動作について知りたい方は筆者の[過去の記事](https://zenn.dev/akfm/articles/next-app-router-client-cache)をご参照ください。
:::

### 任意のタイミングでrevalidate

- `staleTimes`は有効期限を設定するものですが、即座にRouter Cacheを破棄・更新したいということもあるでしょう。
- その場合には、大きく以下3つの方法があります。
  - `router.refresh()`
  - Server Actionsで`revalidatePath()`/`revalidateTag()`
  - Server Actionsで`cookies.set()`/`cookies.delete()`

## トレードオフ

### bf cacheを模倣した挙動

- ブラウザバック時において、Router Cacheに該当するキャッシュがあれば`staleTimes`の期限を超えていても利用されます。
- これはMPAにおけるbf cacheの挙動を模倣したものだと説明されています。
- ただ実際には、React Componentがアンマウント・再マウントされているので、状態は破棄されています。
- これを解消したい場合、、筆者が取り組んでるOSSの[location-state](https://github.com/recruit-tech/location-state)が便利なので、ぜひご検討ください。
