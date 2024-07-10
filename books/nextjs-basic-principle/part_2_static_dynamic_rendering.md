---
title: "static/dynamic rendering"
---

## 背景

Next.jsは2016/10に、SSRをサポートしたReactフレームワークとして公開されました。

https://vercel.com/blog/next

その後、約3年半後の2019年に登場する[v9.3](https://nextjs.org/blog/next-9-3)でSSG、[v9.5](https://nextjs.org/blog/next-9-5)でISRが導入されたことでNext.jsは複数のレンダリングモデルをサポートするフレームワークとなりました。

このように、従来Pages Routerでは**SSR/SSG/ISR**という概念を用いて機能が説明されてきました。

## 設計・プラクティス

App Routerにおいても引き続きSSR/SSG/ISRはサポートされていますが、ドキュメント上ではこれらの用語は使われていません。App Routerは現在、**static rendering**と**dynamic rendering**の2つの概念を用いて多くの機能を説明しています。

- [static rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default): 従来のSSGやISR相当で、**build時やrevalidate実行後**にレンダリング
  - revalidateなし: SSG相当
  - revalidateあり: ISR相当
- [dynamic rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering): 従来のSSR相当で、**リクエストごと**にレンダリング

App Routerのデフォルトはstatic renderingとなっており、dynamic renderingにオプトインする方法は以下の通りです。

- [dynamic functions](https://nextjs.org/docs/app/building-your-application/routing/route-handlers#dynamic-functions)(`cookies()`/`headers()`/...)を利用した場合
- `cache: "no-store"`が指定された`fetch`が含まれる場合
- [Route Segment Config](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config)でオプトインした場合
  - `export const dynamic = 'force-dynamic'`
  - `export const revalidate = 0`
- [unstable_noStore](https://nextjs.org/docs/app/api-reference/functions/unstable_noStore)を利用した場合

Route Segment Configと`unstable_noStore`はdynamic/static renderingを利用者が意識して使うAPIなので明示的ですが、dynamic functionsと`cache: "no-store"`なfetchはstatic/dynamic renderingを意識しづらいAPIなので注意が必要です。ページのごく一部でこれらを利用した結果、ページ全体がdynamic renderingになってしまいパフォーマンス劣化を招く可能性があります。

## 結論

dynamic functionsと`cache: "no-store"`なfetch利用時はstatic/dynamic renderingが切り替わる可能性があることに注意して利用するようにしましょう。

また、可能な限りstatic renderingにしておいた方が耐障害性・パフォーマンスなど多くの面でメリットを得られます。dynamic renderingにする場合には、[Data Cache](https://nextjs.org/docs/app/building-your-application/caching#data-cache)の活用を検討するなどしてパフォーマンスに注意しましょう。

## 例外

特になし。
