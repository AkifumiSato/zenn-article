---
title: "Next.jsで戻る厨を満たすrecoil-sync-next"
emoji: "⏳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

以前、Next.jsのスクロール位置復元について記事を書きました。

https://zenn.dev/akfm/articles/next-js-scroll-restore

(この記事でrecoil-sync-nextついても近々記事を書くと言ってたのに、3ヶ月も空いてしまいました)

この記事でSPAとMPA(Multi Page Application)における、ブラウザバック/フォワード時のスクロール位置復元について言及しました。

- MPAではスクロール位置がブラウザによって復元されることがある(ブラウザの実装に依存)
- SPAではこれらが軽視されがち
- Next.jsでもデフォルトでは復元されない(ChromeでSSGページなど一部条件下では復元される)
- Next.jsでは`experimental.scrollRestoration`を有効にするとスクロール位置をsession storageに保存し復元する

これらと同様に、ブラウザバック/フォワード時のUI復元についても軽視されがちなものの1つです。最近もこの手のUI体験の悪さについて、問題提起がされ話題になりました。

https://rentwi.hyuki.net/?1576010373357965312

ブラウザバック/フォワード時のUI復元についてはSPAに限った話ではなく、JavaScriptで実現してるUIパーツ全般においてあまり対応されていないようにも感じます。代表的なのが上記ツイートの冒頭で言われている無限スクロールやアコーディオンの開閉状態などです。

会社の先輩が自身を指して「戻る厨」と表現していたのを拝借し、本稿ではブラウザバック/フォワード時にUI復元を求める人を「戻る厨」と記載していますが、ブラウザバック/フォワード時のUI復元は多くのWebユーザーが求める一般的な要件だと筆者は考えています。本稿は特に、Next.jsにおけるこのUI復元問題について解決する[recoil-sync-next](https://www.npmjs.com/package/recoil-sync-next)の紹介記事です。

## ブラウザバック時のUI状態の復元

### MPAにおけるUI状態の復元

まずはブラウザバック/フォワード時のUI復元のあり方について考えてみましょう。whatwgではブラウザバック/フォワード時のUI復元についてはどのような仕様が定義されているのでしょう？

https://html.spec.whatwg.org/multipage/browsing-the-web.html#restore-persisted-user-state

> Optionally, update other aspects of entry's document and its rendering, for instance values of form fields, that the user agent had previously recorded in entry's persisted user state.

履歴の実態であるhistory entry内に保存される、`persisted user state`にユーザーエージェント(ここではブラウザ)が復元したいstateを格納することができるようです。ここでは例として、form valueが挙げられています。つまり、UI復元についても以前の記事で挙げた`scroll position data`同様に、ユーザーエージェントによって復元**できる**≒ユーザーエージェント側が何をどこまで復元するか決めることができるようです。

実際にChromeとSafariで雑にhtml作って試してみました。以下はテスト内容を実施後、遷移＋ブラウザバックした時の結果です。

| テスト内容 | Chrome | Safari |
|-----|------|------|
| formに値を入力 | 値が復元される | 値が復元される |
| アコーディオンを開閉 | 復元されない | 復元される |
| scriptタグでconsole出力 | 出力される | 出力されない |

UI状態で何を復元するかはユーザーエージェントによって決められることからも、Chromeではform value以外は復元しない一方、SafariではアコーディオンなどのUI状態も含め復元するようです。

ちなみに、consoleが出力されなかったことからSafariではページload時のJavaScriptを実行をしていないようでしたが、アコーディオンの開閉はできることからイベントハンドラがDomに適切に貼られており、Safariではイベントハンドラなども含めてUI状態を復元しているように見えました(この辺はwhatwgに記載が見つけれらなかった上、推測が入ってるので正確ではないかもしれません。詳しい人教えてください)。

### Next.js(SPA)におけるUI復元

一方でSPAにおいてはどうでしょうか？フレームワークにもよるかもしれませんが、本稿ではNext.jsで考えてみます。

Todo

- Next.jsで実験
  - ブラウザバック時もgSSPなどは実行されるのか


## 構成

- 序文
  - Next.jsでスクロール位置がどうやって復元されるか書いた
  - スクロール位置同様に、UIの復元もMPAではなされる
  - SPAでは軽視されがち
- ブラウザバック時のDomの復元
  - MPA <- ここまで
  - SPA(Next.js)
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
