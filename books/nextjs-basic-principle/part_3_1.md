---
title: "第3.1部 Cache Components"
---

[Cache Components↗︎](https://nextjs.org/docs/app/getting-started/cache-components)はv16で導入されたオプトイン機能で、v14~15で実験的機能だったPPR・Dynamic IO・`"use cache"`などを統合したものです。Cache Componentsは、従来のCache戦略の課題を根本解決すべく再設計されているため、従来のCache戦略と比較して非常に洗練されてると筆者は感じています。

第3.1部では、Cache Componentsについて解説します。

:::message
Cache ComponentsのCache戦略は従来のものとは全く異なるため、第3部とは同テーマながら別物であることを表現するため、第3.1部としています。
:::

:::message
執筆時時点ではオプトインですが、将来的にデフォルトになるかもしれません。
:::
