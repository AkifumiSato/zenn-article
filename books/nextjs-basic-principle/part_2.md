---
title: "第2部 コンポーネント設計"
---

以下RFCのmotivationを見ると、React Server Componentsはより積極的なサーバー活用を目指して生まれたアーキテクチャであることがわかります。

https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md#motivation

第1部で解説してきた通り、特にデータフェッチに関しては従来よりシンプルでセキュアに実装できるようになったことで、データフェッチをコンポーネントにカプセル化することがより容易になりました。

これらを踏まえ、いわゆるコンポーネント設計は従来とは考え方を大きく変えていく必要があります。Server Components・Client Componentsの組み合わせ方、テスト容易性との兼ね合い、並行レンダリングとStreamingの活用など様々な観点を踏まえながら設計を行っていく必要があります。

第2部ではReact Server Componentsにおけるコンポーネント設計パターンを解説します。
