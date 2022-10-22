---
title: "Next.jsで戻る厨を満たすrecoil-sync-next"
emoji: "⏳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---


## 構成

- 序文
  - Next.jsでスクロール位置がどうやって復元されるか書いた
  - スクロール位置同様に、UIの復元もMPAではなされる
  - SPAでは軽視されがち
- ブラウザバック時のDomの復元
  - MPA
  - SPA
  - Next.js
- recoil-sync-next
  - 概要説明
    - URLやhistoryに紐づけてrecoil stateをURLやSession Storageへ保存する
    - これにはNext.jsの実装とrecoil-syncを理解する必要がある
  - recoil
  - recoil-sync
  - Next.js
  - recoil-sync-nextの実装
- 応用編
  - react-hook-formとの連携
  - URL Persistenceの注意点
    - Formの内容などをURLに保存すると危険
  - リロード時
    - Next.jsのバグが原因で対応できない
    - PRは出してるがNested LayoutのRouter実装と衝突する部分があるのか、開きっぱなし
- まとめ
  - 最近ツイッターでも無限スクロールやブラウザバックによる消失問題について議論されてるのを見かけた
  - 個人的にもレコメンドの消失とかは結構辛い
  - これらの体験はrecoil-syncやrecoil-sync-nextによって改善する
  - もっと使って欲しい
  - 使わない人もこれらの体験については改めて考えてみて欲しい
  - よりよい戻る体験が提供されると嬉しい
