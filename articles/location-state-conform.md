---
title: "location-stateをconformに対応させた"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "conform", "nextjs"]
published: false
---

この記事は[location-state](https://www.npmjs.com/package/@location-state/core)を[conform](https://ja.conform.guide/)に対応させるために開発した、[@location-state/conform](https://www.npmjs.com/package/@location-state/conform)というライブラリの紹介記事です。

## ブラウザバック体験の重要性

- このライブラリのモチベーションでもあるブラウザバック体験について、従前記事でも触れていますが大事だと思うので改めて概要だけ触れたいと思います
- Reactを利用している場合、`useState`を使ってるとブラウザバック時に復元されません。
- そのため、アコーディオンやform要素の状態はブラウザバック時にはリセットされてしまいます。
  - 筆者もスマホでformを途中まで入力して、情報を確認するために前のページにブラウザバックして戻ってきたらformが空になってた、という経験があります。
  - 当たり前に消えないと思ってたので、悲しい気持ちになりました。
- これは当然実装にもよるのですが、従来のMPAではbfcacheやブラウザ側の復元処理によってDOMの状態が復元されていました。
- ユーザーはこのような挙動の違いを望まないだろうことが考えられます。
- 実際そういった問題提起がされ話題になることもあります
  - https://rentwi.hyuki.net/?1576010373357965312
- しかしかといって、自前で履歴ごとに復元されるような状態管理を実装するのはかなり大変です。
- この課題を解消すべく開発されたのが、[location-state](https://www.npmjs.com/package/@location-state/core)です。

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
