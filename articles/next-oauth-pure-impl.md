---
title: "Next.jsにOAuthクライアントをライブラリなしで実装する"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "oauth"]
published: false
---

## 構成

- 序文
  - OAuthはアクセストークンの取得を安全に行うためのRFC
  - NextAuthやfastify authなど色々ライブラリはある
  - 今回はあえて、これらのライブラリを用いずにOAuthクライアントをNext.jsに実装する記事です
- OAuth Provider側の設定
  - 今回は準備しやすいであろうGithub OAuthを使います
  - 大抵個人でアカウント持ってるだろうから楽かなと思っただけなので、他でも良いです
  - TBW: Github OAuth app setting
- Next.jsの準備
  - Create Next App
  - セッション管理にRedisをdocker composeで導入します
- OAuthの認可コードフローの実装
  - OAuthプロバイダーに遷移する前に、セッションにCSRF tokenを保存
  - OAuthプロバイダーにstateパラメータ付きでリダイレクト
  - OAuthプロバイダー側で認証
  - callbackのURLでstateの検証
  - token取得
  - userの取得
  - 成功
