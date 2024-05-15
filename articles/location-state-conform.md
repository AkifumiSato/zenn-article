---
title: "@location-state/conformをリリースした"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "conform", "nextjs"]
published: false
---

この記事はlocation-stateをconformに対応させるために開発した、[@location-state/conform](https://www.npmjs.com/package/@location-state/conform)の紹介記事です。

## location-stateとは

location-stateは**履歴位置に同期する状態管理ライブラリ**です。

https://github.com/recruit-tech/location-state

Next.jsなどを採用している場合、ページ内の`useState`は遷移時のunmountで状態が破棄され、ブラウザバック時には**復元されません**。そのため、アコーディオンやform要素の状態はブラウザバック時にはリセットされてしまいます。これはNext.jsに限らず、ReactやVueなどをベースにしたモダンなフロントエンドフレームワークを採用して、クライアントサイドルーティングが発生するSPA遷移の場合に起きがちな挙動です。クライアントサイドルーティングが不在なMPAでは、bfcacheやブラウザ側の復元処理によってDOMの状態が復元されます。

筆者もサイト利用時に、formの入力途中に前のページの情報を確認するためにブラウザバックし、再度formに戻ってきたら入力内容が消えていた経験があります。ユーザーにとって、SPAとMPAでブラウザバック時の挙動が異なることは望ましくありません。

:::message
SPAとMPAの挙動の違いについては、筆者の[過去の記事](https://zenn.dev/akfm/articles/recoi-sync-next#%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%83%90%E3%83%83%E3%82%AF%E6%99%82%E3%81%AEui%E7%8A%B6%E6%85%8B%E3%81%AE%E5%BE%A9%E5%85%83)で詳細に解説しているので、興味がある方はぜひご覧ください。
:::

しかし、開発者が自前で履歴ごとに復元されるような状態管理を実装するのは非常に大変です。これらの課題を解消すべく開発されたのがlocation-stateです。

以下の記事でlocation-stateの詳細について解説しているので、より詳細に知りたい方は以下の記事をご覧ください。

https://zenn.dev/akfm/articles/location-state

## conform

さて、今回はこのlocation-stateがconformに対応したわけなので、conformについても簡単に紹介しておきます。conformは[react-hook-form](https://react-hook-form.com/)などより後発な、Reactのformライブラリです。

https://ja.conform.guide/

主な特徴としては以下が挙げられます。

- validationライブラリとの統合が容易
- 強力なTypeScriptサポート
- Server ActionsやReactのhooksとの親和性が高い
- Progressive Enhancementに対応

筆者はconformを、**Server Actions時代のformライブラリ**として台頭する可能性があると考え、非常に注目しています。以下の記事でより詳細に紹介しているので、興味のある方はぜひご覧ください。

https://zenn.dev/akfm/articles/server-actions-with-conform

## location state conform

ブラウザバック体験を破壊しないようサポートしたいlocation-stateをconformに対応させたのが、今回開発した`@location-state/conform`です。

https://www.npmjs.com/package/@location-state/conform

`@location-state/core`と併用して利用できます。以降は`@location-state/conform`利用前後での挙動の違いや、利用方法について紹介したいと思います。

### @location-state/conformなしでの挙動

- TBW: examplesのキャプチャを貼って詳細に説明

### @location-state/conformを追加・実装

### @location-state/conform導入後の挙動

- TBW: examplesのキャプチャを貼って詳細に説明

## 感想

- 開発中に色々試してて気づいたんですが、conformで作った動的フォームがPEに対応できることに驚きました
- 開発中、formが空になる体験をみてやっぱりかなり辛い体験だなぁと思いました
- この気持ちを減らすべく、もっと多くの人に使ってもらえたら嬉しいです
