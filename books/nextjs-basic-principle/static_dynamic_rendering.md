---
title: "static/dynamic rendering"
---

## 背景

Next.jsは2016/10に、SSRをサポートしたReactフレームワークとして公開されました。

https://vercel.com/blog/next

その後、約3年半後の2019年に登場する[v9.3](https://nextjs.org/blog/next-9-3)でSSG、[v9.5](https://nextjs.org/blog/next-9-5)でISRが導入されたことでNext.jsは複数のレンダリングモデルをサポートするフレームワークとなりました。

このように従来Pages Routerでは、SSR/SSG/ISRという概念を用いて機能が説明されてきました。

## 設計・プラクティス

App Routerにおいても引き続きSSR/SSG/ISRはサポートされていますが、ドキュメント上ではこれらの用語は使われていません。App Routerは現在、**static rendering**と**dynamic rendering**の2つの概念を用いて多くの機能を説明しています。

- [static rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default): 従来の SSG や ISR 相当で、build 時や revalidate 実行後にレンダリング
  - revalidate なし: SSG 相当
  - revalidate あり: ISR 相当
- [dynamic rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering): 従来の SSR 相当で、リクエストごとにレンダリング

App Routerのデフォルトはstatic renderingで、dynamic renderingはオプトインとなっています。オプトインする方法は以下の通りです。

- [dynamic functions](https://nextjs.org/docs/app/building-your-application/routing/route-handlers#dynamic-functions)(`cookies()`/`headers()`/...)を利用した場合
- `cache: "no-store"`が指定された`fetch`が含まれる場合
- [Route Segment Config](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config)でオプトインした場合
  - `export const dynamic = 'force-dynamic'`
  - `export const revalidate = 0`
- [unstable_noStore](https://nextjs.org/docs/app/api-reference/functions/unstable_noStore)を利用した場合

Route Segment Configと`unstable_noStore`はdynamic/static renderingを意識して利用するAPIなので明示的ですが、`dynamic`と`cache: "no-store"`は前提としてdynamic renderingでないと成り立たない、故にこれらを利用する際には自然とdynamic renderingになるという設計のため、利用者は意図せずstatic/dynamic renderingを切り替えてしまう可能性があるので注意が必要です。

## 結論

可能な限りstatic renderingにしておいた方が耐障害性・パフォーマンスなど多くの面でメリットを得られます。Server Componentsがstatic/dynamic renderingどちらなのか、dynamic renderingの場合には[Data Cache](https://nextjs.org/docs/app/building-your-application/caching#data-cache)の活用を意識するなどしてパフォーマンスに注意しましょう。

## 例外

特になし。
