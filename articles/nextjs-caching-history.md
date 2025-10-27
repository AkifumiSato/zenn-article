---
title: "Next.js Cache回想"
emoji: "👨‍💻"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: true
---

:::message
この記事は[JSConf JP 2025](https://jsconf.jp/2025/ja)で発表した[Next.js Caching - Legacy, Improvement, Re-Architecture](https://jsconf.jp/2025/ja/talks/nextjs-caching-re-architecture)を、記事として執筆しなおしたものです。
:::

Next.jsは、「高いパフォーマンスと優れた開発者体験」を提供することを重視したフレームワークです。

## 構成

- 序文
- Next.jsの歴史
- v13: App Router初期のCache
- v14, v15: Cacheの改善
- v16: Cache Components
- 考察: "use cache"に学ぶ抽象化とOSSのプロセス
- v17~:
