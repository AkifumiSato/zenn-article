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

一方でSPAにおいてはどうでしょうか？フレームワークにもよるかもしれませんが、本稿ではNext.jsで考えてみます。Next.jsでは初回のページリクエスト以降の遷移、特に`Link`コンポーネントによる内部遷移はNext.jsのRouterによってJavaScript制御の擬似遷移となります。遷移と同時にページに対応するコンポーネントが画面に描画されます。

formの値などについては`setState`や[react-hook-form](https://react-hook-form.com/)による制御が基本です。Next.jsなどの擬似遷移はページ遷移ごとに元々表示していたコンポーネントをアンマウントするので、多くの場合**これらで保持されている状態は破棄**されます。グローバルな状態管理や`_app.tsx`、新たにNext.jsに導入される[Layout](https://nextjs.org/blog/layouts-rfc)を利用することでページを跨いだ状態管理も可能ですが、これらは履歴に紐づく状態ではなくグローバルな状態のため、以下のような挙動の違いがあります。

| No | アクション      | ブラウザバック時に望ましい復元 | グローバルな状態管理による復元 |
|---|------------|----|---|
| 1 | page Aへ遷移  | -  | - |
| 2 | アコーディオンを開く | アコーディオンは開いてる | アコーディオンは開いてる |
| 3 | page Bへ遷移  | -  | - |
| 4 | page Aへ遷移  | アコーディオンは**閉じている** | アコーディオンは**開いてる** |

グローバルな状態管理はURLや履歴を超えて持ち運べるものなので、当然こうなります。(キーに履歴idを入れれば、と思った方は[こちらへ](https://dic.pixiv.net/a/%E5%90%9B%E3%81%AE%E3%82%88%E3%81%86%E3%81%AA%E5%8B%98%E3%81%AE%E3%81%84%E3%81%84%E3%82%AC%E3%82%AD%E3%81%AF%E5%AB%8C%E3%81%84%E3%81%A0%E3%82%88))

では`getServerSideProps`についてはどうでしょうか？擬似遷移の際、サーバー側から`getServerSideProps`の結果を取得するためにページからは`/_next/data/[hash]/[page].json`のようなfetchリクエストが発生します。ブラウザバック/フォワード時には、このリクエストが毎回飛ぶので、同様に古い`getServerSideProps`の情報は破棄されます。

実験に以下のような`getServerSideProps`を用意しました。

```ts
export const getServerSideProps: GetServerSideProps<Props> = async ({ params }) => {
  const date = new Date()

  return {
    props: {
      miniutes: `${date.getMinutes()}`,
    },
  }
}
```

ページには現在の分数が表示されるようにしておき、`Link`経由で回遊後ブラウザバックすると、現在の分数が取得されました。

## SPAでブラウザバック時に状態を復元するには

SPAでブラウザバック時に状態を復元するには、単純なグローバルな状態管理ではなく履歴に紐づく状態管理が必要です。Next.jsで履歴を一位に特定する方法はあるのでしょうか？Next.jsのRouter内部には履歴を一位に判定する`_key`が存在します。

https://github.com/vercel/next.js/blob/v12.3.2-canary.43/packages/next/shared/lib/router/router.ts#L870

この`_key`は[window.history.state](https://developer.mozilla.org/ja/docs/Web/API/History/state)に`key`として格納されます。

https://github.com/vercel/next.js/blob/v12.3.2-canary.43/packages/next/shared/lib/router/router.ts#L1810

前の記事でも触れましたが、これは元々インクリメンタルなインデックスで、挙動的にもバグになってたのを、筆者が修正プルリク投げて`_key`に変更しました。

https://github.com/vercel/next.js/pull/36861

ドキュメント化された仕様ではないものの、これを利用することで履歴を一意に特定することができます。これも前回書いた通り、実はリロード対応できてないので修正したものの、Nested Layoutに忙しいのかなかなかレビューされません...

https://github.com/vercel/next.js/pull/37127

## recoil-sync-next

さて、履歴が一意に特定できるならあとは状態を履歴ごとに保存すればいいだけです。ここで本校の主題の`recoil-sync-next`が出てくるわけですが、その前にrecoilとrecoil-syncについて軽く触れましょう。

### recoil

### recoil-sync



## 構成

- recoil-sync-next <- ここまで
  - 概要説明
    - URLやhistoryに紐づけてrecoil stateをURLやSession Storageへ保存する
    - これにはNext.jsの実装とrecoil-syncを理解する必要がある
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
