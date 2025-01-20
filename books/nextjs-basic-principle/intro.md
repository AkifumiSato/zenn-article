---
title: "はじめに"
---

Next.jsのPages RouterとApp Routerは、機能・設計・プラクティスなどあらゆる面で大きく異なります。Next.jsはApp Routerへの移行を推奨していますが、本書執筆時点でApp Routerを採用してる事例は日本ではまだまだ少ないこともあり、設計やプラクティスが普及してないと筆者は感じています。

本書は、App RouterやReact Server Componentsの根底にある**考え方**に基づいた設計やプラクティスを、筆者なりにまとめたものです。公式ドキュメントはじめ、筆者や筆者の周りで共有されている理解・前提知識などを元にまとめました。

対象読者は、「App Routerをなんとなく説明できるがちゃんと使ったことはない初学者」〜「実際にApp Routerを用いたアプリケーション開発で苦戦している中級者」を想定しています。できるだけ初学者にもわかりやすい説明になるよう心がけましたが、本書は**入門書ではない**ので、前提知識として説明を省略している部分もあります。

入門書としては[公式のLearn](https://nextjs.org/learn)や[実践Next.js](https://gihyo.jp/book/2024/978-4-297-14061-8)などをお勧めします。

https://nextjs.org/learn

https://gihyo.jp/book/2024/978-4-297-14061-8

本書を通じてNext.jsの考え方を学び、自信を持って実装する方が増えてくれたら幸いです。

:::message alert
本書は執筆時の最新であるNext.js v15系を前提としています。
:::

## 変更履歴

- [2024/08](https://github.com/AkifumiSato/zenn-article/pull/65/files)
  - 初稿
- [2024/10](https://github.com/AkifumiSato/zenn-article/pull/67/files)
  - 追加
    - 第4部
      - [Server Componentsの純粋性](part_4_pure_server_components)
    - 第5部
      - [第5部 その他のプラクティス](part_5)
      - [リクエストの参照とレスポンスの操作](part_5_request_ref)
      - [認証と認可](part_5_auth)
      - [エラーハンドリング](part_5_error_handling)
  - 校正
    - [細粒度のREST API設計](part_1_fine_grained_api_design)
- [2025/01](https://github.com/AkifumiSato/zenn-article/pull/69/files)
  - 更新
    - Next.js v15対応
  - 追加
    - 第3部
      - [[Experimental] Dynamic IO](part_3_dynamicio)
