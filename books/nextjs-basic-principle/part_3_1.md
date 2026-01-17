---
title: "第3.1部 Cache Components"
---

- Cache Componentsはv16で導入された、新たなCache戦略に基づく機能群を有効にする機能フラグ
- v14で発表されたPPRやv15で発表されたDynamic IO・`"use cache"`などを統合し、1つの世界観として整理されたもの
- 執筆時時点では将来的にデフォルトになるのかどうかなどは不明だが、Cache Componentsが従来のCache戦略と比べて非常に洗練された設計であると筆者は感じている

第3.1部では、Cache ComponentsのCache戦略について解説します。

:::message
Cache ComponentsのCache戦略は従来のものとは全く異なるため、第3部とは同テーマながら別物であることを表現するため、第3.1部としています。
:::
