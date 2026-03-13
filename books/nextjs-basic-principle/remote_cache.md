---
title: "Cacheの保存先と共有"
---

## 要約

Next.jsには`"use cache"`以外にも、`"use cache: remote"`や`"use cache: private"`というディレクティブが用意されています。これらはキャッシュの保存先がそれぞれ異なり、`"use cache"`と`"use cache: remote"`は`next.config.ts`などの`cacheHandlers`で保存先を設定することができます。

CacheをRedisなどで永続化・共有したい場合には、`"use cache: remote"`を積極的に活用しましょう。

## 背景

Next.jsに限らず、キャッシュの保存先は重要な要件です。よくあるキャッシュ保存先として以下が挙げられます。

- サーバーのインメモリ
- ローカルファイルシステム
- Redis/Valkey

特に、複数プロセスでキャッシュを共有するような場合にはRedis/Valkeyがよく利用されます。複数プロセスでキャッシュを共有しなくても良いような場合には、インメモリやローカルファイルシステムが採用されることもあります。

## 設計・プラクティス

Next.jsには、`"use cache"`以外にもキャッシュ関連のディレクティブとして`"use cache: remote"`や`"use cache: private"`があります。

これらはそれぞれキャッシュの保存先が異なり、`"use cache"`や`"use cache: remote"`の保存先は、[cacheHandlers↗︎](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheHandlers)で設定することができます。

| ディレクティブ         | 推奨保存先                   | `cacheHandlers`プロパティ名 | 主な用途                                                        |
| :--------------------- | :--------------------------- | :-------------------------- | :-------------------------------------------------------------- |
| `"use cache"`          | インメモリ                   | `default`                   | 単一サーバー内での高速なキャッシュアクセス                      |
| `"use cache: remote"`  | Redis/Valkeyなどリモート環境 | `remote`                    | 複数プロセス間でキャッシュを共有・永続化                             |
| `"use cache: private"` | クライアントサイド（固定）   | （設定不可）                | `cookies()`などに依存する、ユーザー固有のprivateなデータのキャッシュ |

### `"use cache"`

`"use cache"`は、Static ShellにDynamicな処理やコンポーネントを含めることを主なユースケースとして想定していますが、Dynamic Content内の一部から利用することも可能です。

`"use cache"`の保存先は、デフォルトではインメモリとなっており、`next.config.ts`の`cacheHandlers.default`で設定することができます。

```ts :next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheHandlers: {
    default: require.resolve("./cache-handlers/default-handler.js"),
  },
};

export default nextConfig;
```

なお、設定するHandlerのファイルは[`CacheHandler`↗︎](https://github.com/vercel/next.js/blob/87b7ca09d5fdb4ae4207802080b8e629b0e5ebab/packages/next/src/server/lib/cache-handlers/types.ts#L42-L80)を実装し`exports`する必要があります。詳細な実装例は[公式ドキュメントの実装例↗︎](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheHandlers#api-reference%23basic-in-memory-cache-handler)をご参照ください。

:::message alert
インメモリキャッシュは当然ながら複数プロセスで共有できないため、キャッシュの不整合を引き起こす可能性があります。注意しましょう。
:::

### `"use cache: remote"`

`"use cache: remote"`はリモートにある外部Storageにキャッシュを保存することを宣言するディレクティブです。

```ts :next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheHandlers: {
    remote: require.resolve("./cache-handlers/remote-handler.js"),
  },
};

export default nextConfig;
```

:::message alert
`"use cache: remote"`の保存先はデフォルトではインメモリとなっています。`"use cache: remote"`を利用する際には保存先を任意のリモート環境に設定しましょう。
:::

共有キャッシュの永続化にはRedis/Valkeyなどを利用することが一般的ですが、Redis/Valkeyへのアクセスはネットワーク通信を伴うため、インメモリキャッシュより低速になります。そのため、可能な限り`"use cache"`でインメモリキャッシュを活用しつつ、複数プロセスでキャッシュを共有する必要があるケースに限り`"use cache: remote"`を利用する、といった使い分けが望ましいと考えられます。

:::message

- キャッシュヒット率が著しく低い場合、キャッシュするメリットよりコストの方が高くなる可能性があります。
- リモート環境へのアクセスはネットワーク通信を伴うため、インメモリキャッシュより低速になります。

:::

### `"use cache: {name}"`

Cache保存先を複数用意したい場合には、`cacheHandlers`で任意の名前の保存先を設定しておくことで、`"use cache: {name}"`を利用することができます。以下は`"use cache: session"`の例です。

```ts :next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheHandlers: {
    session: require.resolve("./cache-handlers/session-handler.js"),
  },
};

export default nextConfig;
```

このように任意の保存先と名称を設定できることは、キャッシュの保存先を複数使い分けたいようなケースなどで有用です。

### `"use cache: private"`

`"use cache: private"`は、`cookies()`や`headers()`といったprivateなデータに依存するキャッシュを取り扱うためのディレクティブで、他のディレクティブとは異なりクライアントサイドに保存されます。

:::message
`"use cache: private"`の保存先はクライアントサイド固定のため、`cacheHandlers`で設定することはできません。
:::

`"use cache"`や`"use cache: remote"`は、対象の関数やコンポーネント内で`cookies()`や`headers()`を扱うことができません^[引数として受け取ってキャッシュのキーにすることができます]が、`"use cache: private"`は対象の関数やコンポーネント内で`cookies()`や`headers()`を扱うことができます。

```tsx
export default async function getUser() {
  "use cache: private";

  // ✅ `"use cache: private"`なら`cookies()`をコンポーネント内で直接利用可能
  const cookies = await cookies();
  const sessionId = cookies.get("session-id");
  const user = await fetch(`/api/user?sessionId=${sessionId}`);
  return user;
}
```

## トレードオフ

### セルフホスティングでの設定

Vercelなどプラットフォームにデプロイする場合には`cacheHandlers`を設定する必要はありませんが、複数プロセスのセルフホスティングでCacheを共有するには`cacheHandlers`を設定する必要があります^[[Watt↗︎](https://docs.platformatic.dev/docs/reference/wattpm/overview)や[OpenNext↗︎](https://opennext.js.org/)は、`cacheHandlers`の設定を内包しているため、保存先の設定は各種設定ファイルなどに記述する必要がある可能性があります]。

前述の通り、キャッシュを複数プロセスで共有したい場合には`"use cache: remote"`を活用すべきなので、`cacheHandlers.remote`の保存先を任意のRedis/Valkeyなどに設定しましょう。

### Vercelの`cacheHandlers`

Vercelでは、Next.jsをサーバーレス環境へデプロイするためリクエスト間でインメモリキャッシュを共有・永続化することはできません。

詳細な仕様や背景については[こちらのコメント↗︎](https://github.com/vercel/next.js/issues/85240#issuecomment-3560124078)をご参照ください。
