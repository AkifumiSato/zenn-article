---
title: "Cacheの保存先"
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

`cacheHandlers.default`はデフォルトでインメモリCacheとなっているため、複数プロセスでCache共有ができません。Cacheの共有にはRedisなどを利用することが一般的ですが、Redisはネットワーク通信を伴うためインメモリCacheより低速になります。そのため、`cacheHandlers.default`をRedisに設定する前にまずは次の`cacheHandlers.remote`をRedisに設定し、[`"use cache: remote"`](#use-cache-remote)を活用することを検討しましょう。

::::details Vercelにおける`"use cache"`の挙動
VercelではNext.jsをサーバーレス環境へデプロイするため、`default`のインメモリCacheはリクエストごとに破棄されます。Static ShellはCDNに保存されるためrevalidateが可能ですが、Dynamic Content内の`"use cache"`は実質利用できないとされています。詳しくは[こちらのissue↗︎](https://github.com/vercel/next.js/issues/85240#issuecomment-3560124078)をご参照ください。

ただし、実際には[Dynamic APIsを利用しなければCacheできる↗︎](https://zenn.dev/link/comments/d578f971c18c22)との情報もあり、仕様が明確でないので、利用時には各々挙動を確認することをお勧めします。
::::

### `"use cache: remote"`

`"use cache: remote"`は文字通り、CacheをRedisなどのリモート環境に保存することを想定したディレクティブです。Next.jsの設計意図としては、Runtime dataなどに依存する高カーディナリティ^[カーディナリティ: データのバリエーションのこと。データのバリエーションが多い=高カーディナリティ]なデータに対しては`"use cache: remote"`を利用し、静的に解決できるものやパターンが非常に少ない場合には`"use cache"`を利用することを想定しています。

`"use cache"`はStatic Shellに動的処理やコンポーネントを含めることを主なユースケースとして想定^[参考: [Runtime caching considerations↗︎](https://nextjs.org/docs/app/api-reference/directives/use-cache#runtime-caching-considerations)]していますが、Dynamic Content内の一部から利用することも可能です。ただし、高カーディナリティなデータを扱う場合にはCacheミスの可能性が高くなるため、Next.jsは`default`と`remote`2つの保存先を変更できるようになっています。

そのため、`"use cache: remote"`は同様にDynamic Content内の一部から利用することも可能ですが、より高カーディナリティなデータを扱うことを想定しています。

- 文字通り、リモートにあるCache保存先を選択したい場合に利用することを想定したディレクティブ
- 永続化先としてRedisなどを用意して参照する
- ネットワーク通信は低速なため、`"use cache"`でローカルに保存・参照する方が高速

### `"use cache: private"`(experimental)

- クライアント側に保存されるため、Privateな永続化先を指してることを表すディレクティブ
- 主にユーザー単位データの保存などに利用されることを想定してると考えられる
- experimentalなのは機能開発の途中だからだと思われる

### `"use cache: {name}"`

TBW: `"use cache: {name}"`のように、カスタムすることもできるが、代表的なユースケースをフォローするために`"use cache: remote"`や`"use cache: private"`が用意されてる

### CacheHandler

- Cacheを複数プロセスで共有するには、`"use cache: remote"`が適切
  - https://github.com/vercel/next.js/issues/85240#issuecomment-3560124078
- セルフホスティングでは`remote`の設定から始めるのが良さそう
- 複数の保存先を設定したい場合、`remote`以外に`session`など独自の命名を追加することも可能
- wattがCacheHandlersを内包してたりする
  - ただし執筆時現在、`default`しか設定されてない
- （OpenNextについても要調査）

## トレードオフ

### プラットフォーム依存なCache制御

- VercelのServerless Functionsでは、Serverlessなので当然ながらインメモリなCacheは扱えない。つまり`"use cache"`を宣言してもCacheされないように見えることがある
  - 仕組みは不明だが、Static Shellは共有できてるらしい
    - https://x.com/108yen___/status/2007466546906714345?s=20
    - [CDNに保存してるらしい](https://github.com/vercel/next.js/discussions/88136#discussioncomment-15423898)が、、明記されてない
- セルフホスティングにおけるマルチプロセスでは、`"use cache: remote"`+`updateTag(tag)`の積極的利用を考えるべき
  - `"use cache"`はプロセス単位のCacheとして、revalidateが不要もしくはCache不整合が許容できるような場合のみ利用するのが良さそう
