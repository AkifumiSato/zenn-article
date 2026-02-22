---
title: "Cacheのディレクティブと保存先"
---

## 要約

Cacheの保存先は`next.config.ts`の`cacheHandlers`で設定することができ、`default`、`remote`、もしくは任意の名前の保存先を設定することが可能です。`"use cache"`は`default`に、`"use cache: remote"`は`remote`に、`"use cache: private"`はクライアントサイドで保存されます。

特に`"use cache"`は、Static Shellに含めるかインメモリに保存されるので、Redisなどで永続化する場合には`"use cache: remote"`を積極的に活用しましょう。

## 背景

一般的にCacheを扱う場合、保存先は重要な要件です。よくあるCache保存先として以下が挙げられます。

- サーバーのインメモリ
- ローカルファイルシステム
- Redis

Next.jsにおいてもCacheを扱う以上これらに保存したいことは多々あります。またユースケースに応じて複数保存先を用意することも考えられます。

特にNext.jsをセルフホスティングで扱う場合、Cacheの保存先について特に設定などもなく実装すると、Cacheの不整合や予期せぬ動作を引き起こす可能性があります。

## 設計・プラクティス

Next.jsには、`"use cache"`以外にもCache関連のディレクティブとして`"use cache: remote"`や`"use cache: private"`があります。

これらはそれぞれCacheの保存先が異なり、`"use cache"`や`"use cache: remote"`の保存先は、[cacheHandlers↗︎](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheHandlers)で設定することができます。

:::message
`"use cache"`の保存先はデフォルトではインメモリのため、特にセルフホスティングではCache不整合の原因になりえるので注意が必要です。
:::

:::message
`"use cache: private"`はクライアントサイドで保存されるため、cacheHandlers↗で設定することはできません。
:::

### `"use cache"`（`"use cache: default"`）

`"use cache"`の保存先は`cacheHandlers.default`で設定することができます。

```ts :next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheHandlers: {
    default: require.resolve("./cache-handlers/default-handler.js"),
  },
};

export default nextConfig;
```

`cacheHandlers.default`はデフォルトでインメモリCacheとなっているため、複数プロセスでCache共有ができません。Cacheの共有にはRedisなどを利用することが一般的ですが、Redisはじめリモート環境にCacheを保存するにはネットワーク通信を伴うため、インメモリCacheより低速になります。そのため、`cacheHandlers.default`をRedisに設定する前にまずは次の`cacheHandlers.remote`をRedisに設定し、[`"use cache: remote"`](#use-cache-remote)を活用することを検討しましょう。

### `"use cache: remote"`

`"use cache: remote"`は文字通り、CacheをRedisなどのリモート環境に保存するためのディレクティブです。`"use cache"`はCDNやインメモリに保存されることを想定して設計されているので、セルフホスティングやサーバーレス環境でCacheを共有したい場合には`"use cache: remote"`を利用しましょう。

`"use cache: remote"`は`cacheHandlers.remote`で設定することができます。

```ts :next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheHandlers: {
    remote: require.resolve("./cache-handlers/remote-handler.js"),
  },
};

export default nextConfig;
```

ただし、`"use cache: remote"`はリモート環境にCacheを保存するため、いくつかの点で注意が必要です。

- ネットワーク通信が伴うため、インメモリCacheより低速になります。
- Cacheヒット率が著しく低い場合、Cacheするメリットよりコストの方が高くなる可能性があります。

### `"use cache: private"`(experimental)

`"use cache: private"`は、`cookies()`や`headers()`といったprivateなデータに依存するCacheを取り扱うためのディレクティブで、ユーザーのprivate空間であるクライアントサイドに保存されます。そのため、`"use cache: private"`は`cacheHandlers`で設定することはできません。

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

### セルフホスティングでの設定

Vercelなどプラットフォームにデプロイする場合には`cacheHandlers`は設定済みのため、特に設定する必要はありません。一方、セルフホスティングでは`cacheHandlers`を自前で設定する必要があります^[[Watt↗︎](https://docs.platformatic.dev/docs/reference/wattpm/overview)や[OpenNext↗︎](https://opennext.js.org/)は、`cacheHandlers`の設定を内包しているため、保存先の設定は各種設定ファイルなどに記述する必要がある可能性があります]。

前述の通り、`"use cache"`はStatic Shellへの追加やインメモリCacheとして利用されることを想定しているため、基本的には`cacheHandlers.default`はデフォルトのままにしておくことが望ましいと考えられます。整合性が担保されたCacheが必要な場合には、まずは`cacheHandlers.remote`の保存先を任意のRedisなどに設定して`"use cache: remote"`を活用することを検討しましょう。

## トレードオフ

### プラットフォームの`cacheHandlers`

プラットフォーム側の構成や`cacheHandlers`の設定の詳細は、各プラットフォームに依存するためブラックボックスな可能性があります。

VercelではNext.jsをサーバーレス環境へデプロイするため、`default`のインメモリCacheはリクエストごとに破棄されます。Static ShellはCDNに保存されるためrevalidateが可能ですが、Dynamic Content内の`"use cache"`は実質利用できないとされています。詳細な仕様や背景については[こちらのコメント↗︎](https://github.com/vercel/next.js/issues/85240#issuecomment-3560124078)をご参照ください。

ただし、実際には[Dynamic APIsを利用しなければCacheできる↗︎](https://zenn.dev/link/comments/d578f971c18c22)との情報もあり、仕様が明確でないので、利用時には各々挙動を確認することをお勧めします。
