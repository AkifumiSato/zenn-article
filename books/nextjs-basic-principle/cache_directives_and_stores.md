---
title: "Cacheのディレクティブと保存先"
---

## 要約

Next.jsには`"use cache"`以外にも、`"use cache: remote"`や`"use cache: private"`というディレクティブが用意されています。これらはそれぞれCacheの保存先が異なり、`"use cache"`と`"use cache: remote"`は`next.config.ts`の`cacheHandlers`で保存先を設定することができます。

CacheをRedisなどで永続化・共有したい場合には、`"use cache: remote"`を積極的に活用しましょう。

## 背景

Next.jsに限らず、サーバー側でCacheを扱う場合に保存先は重要な要件です。よくあるCache保存先として以下が挙げられます。

- サーバーのインメモリ
- ローカルファイルシステム
- Redis

特にセルフホスティングにおいては複数サーバーでCacheを共有するために、Redisがよく利用されます。複数サーバーでCacheを共有しなくても良いような場合には、インメモリやローカルファイルシステムが採用されることもあります。

## 設計・プラクティス

Next.jsには、`"use cache"`以外にもCache関連のディレクティブとして`"use cache: remote"`や`"use cache: private"`があります。

これらはそれぞれCacheの保存先が異なり、`"use cache"`や`"use cache: remote"`の保存先は、[cacheHandlers↗︎](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheHandlers)で設定することができます。

| ディレクティブ         | 保存先                | `cacheHandlers` | 主な用途                                                        |
| :--------------------- | :-------------------- | :-------------- | :-------------------------------------------------------------- |
| `"use cache"`          | インメモリ            | `default`       | 単一サーバー内での高速なデータアクセス                          |
| `"use cache: remote"`  | Redisなどリモート環境 | `remote`        | 複数サーバー間でCacheを共有・永続化                             |
| `"use cache: private"` | クライアントサイド    | （設定不可）    | `cookies()`などに依存する、ユーザー固有のPrivateなデータをCache |

### `"use cache"`

`"use cache"`は、Static Shellに動的処理やコンポーネントを含めることを主なユースケースとして想定していますが、Dynamic Content内の一部から利用することも可能です。この場合の保存先は、デフォルトではインメモリとなっています。

```ts :next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheHandlers: {
    default: require.resolve("./cache-handlers/default-handler.js"),
  },
};

export default nextConfig;
```

:::message alert
インメモリCacheは複数サーバーでCache不整合を引き起こすので、注意しましょう。
:::

### `"use cache: remote"`

`"use cache: remote"`はRedisなどリモート環境にCacheを保存することを宣言するディレクティブです。

```ts :next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheHandlers: {
    remote: require.resolve("./cache-handlers/remote-handler.js"),
  },
};

export default nextConfig;
```

Cacheの共有にはRedisなどリモート環境を利用することが一般的ですが、リモート環境へのアクセスはネットワーク通信を伴うため、インメモリCacheより低速になります。そのため、可能な限り`"use cache"`でインメモリCacheを活用しつつ、リモート環境でCacheを共有する必要があるケースにおいては`"use cache: remote"`を利用するような使い分けが好ましいと考えられます。

:::message
Cacheヒット率が著しく低い場合、Cacheするメリットよりコストの方が高くなる可能性があります。
:::

### `"use cache: private"`(experimental)

`"use cache: private"`は、`cookies()`や`headers()`といったprivateなデータに依存するCacheを取り扱うためのディレクティブで、ユーザーのprivate空間であるクライアントサイドに保存されます。

:::message
`"use cache: private"`の保存先はクライアントサイド固定となっており、`cacheHandlers`で設定することはできません。
:::

`"use cache"`や`"use cache: remote"`は、対象の関数やコンポーネント内で`cookies()`や`headers()`を扱うことができませんが、引数として受け取ってCacheのキーにすることができます。一方`"use cache: private"`は、対象の関数やコンポーネント内で`cookies()`や`headers()`を扱うことができます。

```tsx
export default async function getUser() {
  "use cache: private";

  // ✅ "use cache"や"use cache: remote"では`cookies()`は利用不可
  const cookies = await cookies();
  const sessionId = cookies.get("session-id");
  const user = await fetch(`/api/user?sessionId=${sessionId}`);
  return user;
}
```

### `"use cache: {name}"`

Cache保存先を複数用意したい場合には、`cacheHandlers`で任意の名前の保存先を設定しておくことで、`"use cache: {name}"`を利用することができます。

```ts :next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheHandlers: {
    session: require.resolve("./cache-handlers/session-handler.js"),
  },
};

export default nextConfig;
```

Cacheの保存先であるRedisのインスタンスを複数使い分けたいようなケースなどで有用と考えられます。

## トレードオフ

### セルフホスティングでの設定

Vercelなどプラットフォームにデプロイする場合には`cacheHandlers`を設定する必要はありません。一方、セルフホスティングでは`cacheHandlers`を自前で設定する必要があります^[[Watt↗︎](https://docs.platformatic.dev/docs/reference/wattpm/overview)や[OpenNext↗︎](https://opennext.js.org/)は、`cacheHandlers`の設定を内包しているため、保存先の設定は各種設定ファイルなどに記述する必要がある可能性があります]。

前述の通り、`"use cache"`はStatic Shellへの追加やインメモリCacheとして利用されることを想定しているため、基本的には`cacheHandlers.default`はデフォルトのままにしておくことが望ましいと考えられます。整合性が担保されたCacheが必要な場合には、まずは`cacheHandlers.remote`の保存先を任意のRedisなどに設定して`"use cache: remote"`を活用することを検討しましょう。

### プラットフォームの`cacheHandlers`

プラットフォーム側の構成や`cacheHandlers`の設定の詳細は、各プラットフォームに依存するためブラックボックスな可能性があります。

VercelではNext.jsをサーバーレス環境へデプロイするため、`default`のインメモリCacheはリクエストごとに破棄されます。Static ShellはCDNに保存されるためrevalidateが可能ですが、Dynamic Content内の`"use cache"`は実質Cacheされません。

詳細な仕様や背景については[こちらのコメント↗︎](https://github.com/vercel/next.js/issues/85240#issuecomment-3560124078)をご参照ください。
