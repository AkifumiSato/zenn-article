---
title: "第3.1部 Cache Components"
---

[Cache Components↗︎](https://nextjs.org/docs/app/getting-started/cache-components)は、Next.js v16で導入された機能フラグで、PPR・Dynamic IO・`"use cache"`などを統合し、1つの世界観として整理したものです。執筆時時点では将来的にデフォルトになるのかどうかは不明ですが、Cache Componentsは従来のCache戦略と比べて非常に洗練された設計であると筆者は感じています。

第3.1部では、Cache ComponentsのCache戦略について解説します。

:::message
Cache ComponentsのCache戦略は従来のものとは全く異なるため、第3部とは同テーマながら別物であることを表現するため、第3.1部としています。
:::
