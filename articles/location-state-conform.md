---
title: "location-stateをconformに対応させた"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "conform", "nextjs"]
published: false
---

この記事は[location-state](https://www.npmjs.com/package/@location-state/core)を[conform](https://ja.conform.guide/)に対応させるために開発した、[@location-state/conform](https://www.npmjs.com/package/@location-state/conform)というライブラリの紹介記事です。

## location-stateとは

location-stateは**履歴位置に同期する状態管理ライブラリ**です。

Next.jsを採用している場合、ページ内の`useState`は遷移時のunmountで状態が破棄され、ブラウザバック時には**復元されません**。そのため、アコーディオンやform要素の状態はブラウザバック時にはリセットされてしまいます。これはNext.jsに限らず、ReactやVueなどをベースにしたモダンなフロントエンドフレームワークを採用して、クライアントサイドルーティングが発生する場合に起きがちな挙動です。逆にクライアントサイドルーティング処理が不在なMPAでは、bfcacheやブラウザ側の復元処理によってDOMの状態が復元されます。

筆者もユーザーとして、formの入力途中に情報を確認するために前のページにブラウザバック・再度formまで戻ってきたらform要素の入力が空になってた、という経験があります。また、そういった問題提起が話題になったこともあります。

https://rentwi.hyuki.net/?1576010373357965312

ユーザーはこのような挙動の違いを望まないだろうことが考えられます。しかし、自前で履歴ごとに復元されるような状態管理を実装するのはかなり大変です。これらの課題を解消すべく開発されたのが[location-state](https://github.com/recruit-tech/location-state)です。

より詳しくはリリース時に書いた以下の記事をご参照ください。

https://zenn.dev/akfm/articles/location-state

### 従来のSPAとMPAの挙動の比較

- ちなみに、従来のMPAとモダンなSPAフレームワークの挙動の比較や考察は過去に行っているので、詳細はぜひ以下の記事を参照ください。
  - https://zenn.dev/akfm/articles/recoi-sync-next#%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%83%90%E3%83%83%E3%82%AF%E6%99%82%E3%81%AEui%E7%8A%B6%E6%85%8B%E3%81%AE%E5%BE%A9%E5%85%83

## conform

- ブラウザバック体験の話からは打って変わり、昨今のNext.jsとfromライブラリの動向についても触れておきたいと思います
- 上記では「モダンなSPAフレームワーク」と表現しましたが、筆者はNext.jsを使うことが多いです。
- 筆者はApp Routerの登場以降、formライブラリとしてreact-hook-formを使い続けるべきか悩んでいたが、Server ActionsやProgressive Enhancementと相性のいい[conform](https://ja.conform.guide/)が登場して、筆者は特に注目してる
- これらについても以下の記事で別途紹介しているのでよければご参照ください
  - https://zenn.dev/akfm/articles/server-actions-with-conform

## location state conform

- ブラウザバック体験を破壊しないようサポートしたいlocation-stateを、conformに対応させたのが[@location-state/conform](https://www.npmjs.com/package/@location-state/conform)です
- TBW: 使い方

## 感想

- 開発中に色々試してて気づいたんですが、conformで作った動的フォームがPEに対応できることに驚きました
- 開発中、formが空になる体験をみてやっぱりかなり辛い体験だなぁと思いました
- この気持ちを減らすべく、もっと多くの人に使ってもらえたら嬉しいです
