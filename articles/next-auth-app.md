---
title: "Nextと認証"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

## 構成メモ

- introduction
  - Next ＋ Nest で認証作った時のメモ
  - vercel でやりたかったので認証の検証などは全部 API 任せにしたかった
- 認証周りの用語と知識
  - 認証と認可
  - Oauth2
    - passport についても触れる
  - Open id connect
  - jwt について
  - cross origin について
- 実際の処理
  - 処理フロー
    - （図）
    - API へアクセス
    - Cookie 付与
    - 次回からログイン情報取れる
  - cookie
    - http only
    - same site
    - secure
