---
title: "ReactのStateには時間軸が存在する"
emoji: "⌚️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: false
---

ReactはじめSPAのStateは大きく3種類、ローカルState・グローバルState・サーバーStateの3種類、これでおおよそのStateの分類が可能であると考えていました。これに対し会社の先輩から「参照スコープだけでなく、時間軸におけるグローバルなStateもあるよね」という意見をもらい、非常に納得しました。この時間軸スコープをRustからライフタイムという言葉を借りて**Stateライフタイム**と呼称し、Stateの分類を自分なりに考察したいと思います。

## 参照スコープ軸の分類

まずこれまで自分が認識してた参照スコープにおけるStateの分類について触れておきます。以下の3種類で分類可能と考えていました。

- **Local State**: これは`useState`を利用することとほぼ同義です。コンポーネントがアンマウントされるまで生存する状態のことです。
- **Global State**: 複数のコンポーネントから参照されうるStateです。ReduxやRecoil、Context APIなどを通じて管理されることが多いかと思います。 
- **Server State**: クライアントサイドでキャッシュしてるAPIサーバーからのレスポンス値などです。SWRやReact Queryの内部で管理されてるものなどを指します。

細かい用語は違えど、以下の参考リンクではそれぞれ同じよう分類を行っています。

https://zenn.dev/yoshiko/articles/607ec0c9b0408d

以下のリンクでは上記と同様の分類か、データカテゴリによる分類（Data,Control/UI,Session,Communication,Locationの5つ）が可能とされています。

https://react-community-tools-practices-cheatsheet.netlify.app/state-management/overview/#types-of-state

Remixのコントリビュータのブログでは、Local/Globalなどの分類ではなく「全てのStateはUI Stateかサーバーキャッシュの2つに分けられる」と主張しています。

https://kentcdodds.com/blog/application-state-management-with-react#server-cache-vs-ui-state

これらの参考記事は非常に勉強になるので時間のある方はそれぞれ一読することをお勧めします。ここで主張したいこととしては、多くの場合Stateは参照スコープやデータカテゴリによってStateを分類しているということです。

ここに冒頭にあったように時間軸という観点を追加すると、Global Stateとまとめられているものには**生存期間が異なるものが混在している**ことがわかります。

## Stateライフタイム

e.g. アコーディオンの開閉

### Component mount

### Javascript memory

### Browser history

- Navigation API

### Storage

- session
- local

### Expire

## State Managementライブラリを分類

## まとめ

Todo

- 図で主要ライブラリを分類
