---
title: "Next.jsで戻る厨を満たすrecoil-sync-next"
emoji: "⏳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: true
---

以前、Next.jsのスクロール位置復元について記事を書きました。

https://zenn.dev/akfm/articles/next-js-scroll-restore

上記記事でSPAとMPA(Multi Page Application)における、ブラウザバック/フォワード時のスクロール位置復元について言及しました。

- MPAではスクロール位置がブラウザによって復元されることがある(ブラウザの実装に依存)
- SPAではこれらが軽視されがち
- Next.jsにおいても、デフォルトでは復元されない(ChromeでSSGページなど一部条件下では復元される)
- Next.jsでは`experimental.scrollRestoration`を有効にするとスクロール位置をsession storageに保存し復元する

これらと同様に、ブラウザバック/フォワード時のUI復元についても軽視されがちなものの1つです。最近もこの手のUI体験の悪さについて、問題提起がされ話題になりました。

https://rentwi.hyuki.net/?1576010373357965312

ブラウザバック/フォワード時のUI復元についてはSPAに限った話ではなく、JavaScriptで実現してるUIパーツ全般においてあまり対応されていないようにも感じます。代表的なのが上記ツイートの冒頭で言われている無限スクロールやアコーディオンの開閉状態などです。

会社の先輩が自身を指して「戻る厨」と表現していたのを拝借し、本稿のタイトルではブラウザバック/フォワード時にUI復元を求める人を「戻る厨」と記載していますが、ブラウザバック/フォワード時のUI復元は多くのWebユーザーが求める一般的な要件だと筆者は考えています。本稿はSPAとMPAにおけるこの問題の解説と、Next.jsにおけるUI復元問題を解決する[recoil-sync-next](https://www.npmjs.com/package/recoil-sync-next)を紹介記事です。

## ブラウザバック時のUI状態の復元

### MPAにおけるUI状態の復元

まずはブラウザバック/フォワード時のUI復元のあり方について考えてみましょう。whatwgではブラウザバック/フォワード時のUI復元についてはどのような仕様が定義されているのでしょう？

https://html.spec.whatwg.org/multipage/browsing-the-web.html#restore-persisted-user-state

> Optionally, update other aspects of entry's document and its rendering, for instance values of form fields, that the user agent had previously recorded in entry's persisted user state.

履歴の実態であるhistory entry内に保存される、`persisted user state`にユーザーエージェント(ここではブラウザ)が復元したいstateを格納することができるようです。ここでは例として、form valueが挙げられています。つまり、UI復元についても以前の記事で挙げた`scroll position data`同様に、ユーザーエージェントによって復元**できる**≒ユーザーエージェント側が何をどこまで復元するか決めることができるようです。

実際にChromeとSafariで試してみました。以下はローカルサーバーから`Cache-Control: no-store`ヘッダー付きで静的ファイルを返却し、テスト内容を実施後遷移＋ブラウザバックした時の結果です。

| テスト内容 | Chrome | Safari |
|-----|------|------|
| formに値を入力 | 値が復元される | 値が復元される |
| アコーディオンを開閉 | 復元されない | 復元される |

UI状態で何を復元するかはユーザーエージェントによって決められることからも、Chromeではform value以外は復元しない一方、SafariではアコーディオンなどのUI状態も含め復元するようです。

一方、`Cache-Control: no-store`を外すとChromeでもアコーディオン含め再現されました。ChromeやSafariでは一部条件下において、JavaScriptヒープまで含めてDomを再現する[bf cache(back forward cache)](https://web.dev/i18n/ja/bfcache/#bfcache%E3%81%AE%E5%9F%BA%E6%9C%AC)が利用されることがあります。ChromeとSafariではbf cacheが利用される条件が違うため、`Cache-Control`ヘッダーによって挙動に違いが生じたものと考えられます。

### Next.js(SPA)におけるUI復元

一方でSPAにおいてはどうでしょうか？フレームワークにもよるかもしれませんが、本稿ではNext.jsで考えてみます。Next.jsでは初回のページリクエスト以降の遷移、特に`Link`コンポーネントによる内部遷移はNext.jsのRouterによってJavaScript制御の擬似遷移となります。擬似遷移が発生するとページに対応するコンポーネントが画面に描画されます。

formの値などについては`setState`や[react-hook-form](https://react-hook-form.com/)で制御することも多いかと思われます。Next.jsの擬似遷移はページ遷移ごとに元々表示していたコンポーネントをアンマウントするので、多くの場合**これらで保持されている状態は破棄**されます。グローバルな状態管理や`_app.tsx`に配置した`Context`による状態、新たにNext.jsに導入される[Layout](https://nextjs.org/blog/layouts-rfc)を利用することでページを跨いだ状態管理も可能ですが、これらは履歴に紐づく状態ではなくグローバルな状態のため、他履歴entryの状態にも影響してしまいます。

以下はアコーディオンComponentを持つpage Aとpage Bを回遊した際の、アコーディオンの挙動です。アコーディオンComponentは開閉状態を`isOpen: boolean`という形でグローバルに状態管理しているものとします。

| No | アクション              | 望ましい状態 | 実際の挙動    |
|--|--------------------|----------------|------------------|
| 1 | page Aへ遷移          | 閉じてる   | 閉じてる     |
| 2 | アコーディオンを開く         | 開いてる   | 開いてる     |
| 3 | page Bへ遷移          | 閉じてる   | **開いてる** |
| 4 | page Aへ遷移          | 閉じている  | **開いてる** |
| 5 | アコーディオンを閉じる        | 閉じている   | 閉じている     |
| 6 | ブラウザバック2回(page Aへ) | 開いてる  | **閉じている** |

グローバルな状態管理では、アコーディオンの状態が他履歴entryへ影響している様子がみてとれます。

### MPAとSPAの違いまとめ

一旦ここまでのまとめです。ブラウザはスクロール位置の復元同様に、UI状態の復元についてもある程度よしなに行ってくれます。ブラウザごとにある程度条件はあれど、**MPAは基本的にその恩恵を受けやすい傾向**にあります。一方でSPAの場合、擬似遷移をJavaScriptで実装しているため同様の体験を得るためには開発者が自前で実装する必要がありますが、実際に開発者が自前でこの問題を解決するような実装をしてるケースは少ないようにも感じられます。

そのため、**相対的にSPAは体験が悪いと受け止められる**可能性がありますし、残念ながらある程度これは事実かと思います。

## SPAでブラウザバック時に状態を復元するには

SPAでブラウザバック時に状態を復元するには、履歴をkeyとするグローバルな状態管理が必要です。

Next.jsで履歴を一位に特定する方法はあるのでしょうか？Next.jsのRouter内部には履歴を一意に判定する`_key`が存在します。

https://github.com/vercel/next.js/blob/v12.3.2-canary.43/packages/next/shared/lib/router/router.ts#L870

この`_key`は[window.history.state](https://developer.mozilla.org/ja/docs/Web/API/History/state)に`key`として格納されます。

https://github.com/vercel/next.js/blob/v12.3.2-canary.43/packages/next/shared/lib/router/router.ts#L1810

前の記事でも触れましたが、これは元々インクリメンタルなインデックスで、挙動的にもバグになってたのを筆者が修正プルリクを投げて`_key`に変更しました。

https://github.com/vercel/next.js/pull/36861

ドキュメント化された仕様ではないものの、これを利用することで履歴を一意に特定することができます。これも前回書いた通り、実はリロード対応できてないので修正したものの、Nested Layoutに忙しいのかなかなかレビューされないため、現在もリロード時にはスクロール位置やこのkeyは初期化されてしまいます。

https://github.com/vercel/next.js/pull/37127

## recoil-sync-next

`window.history.state.key`を参照することでNext.jsで履歴を一意に特定するできそうです。となると、あとは状態を履歴ごとに保存すればいいだけです。これを容易にするのが、**recoil-sync-next**です。recoilやrecoil-syncについて軽く触れつつ、recoil-sync-nextの利用方法を見ていきましょう。

### recoil

[recoil](https://recoiljs.org)はMetaが開発したReactの状態管理ライブラリです。こちらについては既にご存知の方も多いと思うので、ここでは特徴紹介に留めます。詳細な紹介は他の記事や公式を参照ください。

- コード分割可能なグローバル状態管理ライブラリである
- `export`せずに状態を宣言することで、Component単位のグローバル状態を宣言できる
- 最も基本的な使い方は`useState`同様setter/getterで状態を変更するものである
- 非同期Stateを簡単に定義できる(Suspenseを使う・使わないも選択できる)

### recoil-sync

[recoil-sync](https://recoiljs.org/docs/recoil-sync/introduction/)はrecoilの状態を外部Storeに保存することを容易にするライブラリです。`useEffect`やrecoilの`selector`を利用することなく状態を保存できるのが特徴です。具体的にはatom宣言時にオプションで、`effects`に`syncEffect`のインスタンスを指定します。

```ts
import { number } from '@recoiljs/refine'
import { atom } from 'recoil'
import { syncEffect } from 'recoil-sync'

// ...snip...

const countState = atom<number>({
  key: 'Count',
  default: 0,
  // ↓違いはここのみ
  effects: [
    syncEffect({
      storeKey: 'storeA',
      refine: number(),
    }),
  ],
});
```

利用者側はこれだけです。`syncEffect`にはいくつかのオプションがあり、`storeKey`は保存するStoreを示すキーです。実際の保存ロジックを実装する`RecoilSync`コンポーネントとキーを一致させる必要があります。`refine`は同じくrecoilから提供されているバリデーションライブラリです。これを用いることで予期せぬデータ型の混入を防げます。

また、実際の保存は`_app.tsx`で呼び出す[RecoilSync](https://recoiljs.org/docs/recoil-sync/api/RecoilSync/)コンポーネントのpropsでそれぞれ実装します。例えば`read`は以下のようキーを受け取り、キーをもとにlocal storageなどの外部Storeから値抽出し返ます。

```tsx
const read: ReadItem = useCallback((itemKey) => {
  const storage = JSON.parse(localStorage.getItem('hoge'))
  return storage?.[itemKey] ?? new DefaultValue()
}, [])
// <RecoilSync
//   read={read}
// ...
// >
```

### recoil-sync-next

[recoil-sync-next](https://github.com/recruit-tech/recoil-sync-next)は**Next.jsのhistory keyを用いて状態をsession storageやURLに保存するライブラリ**です。

ここまでrecoil-syncやNext.jsのhistory keyについて解説したものの、これらを意識せずに利用し始めることができ、**履歴ごとの状態の復元を実現します**。例として、session storageに保存する場合の利用方法は以下のようになります。

```tsx
// _app.tsx
import { RecoilHistorySyncJSONNext } from 'recoil-sync-next'

function MyApp({ Component, pageProps }: AppProps) {
  return (
    <RecoilRoot>
      <RecoilHistorySyncJSONNext storeKey="ui-state">
        <Component {...pageProps} />
      </RecoilHistorySyncJSONNext>
    </RecoilRoot>
  )
}

// Recoil State
export const counter = initializableAtomFamily<number, string>({
  key: 'counterState',
  effects: [
    syncEffect({
      storeKey: 'ui-state',
      refine: number(),
    }
  )],
})
```

:::message
2022/10に行われたNext Confでbetaリリースが発表された[app directory](https://nextjs.org/blog/next-13#app-directory-beta)は先述の[Layout](https://nextjs.org/blog/layouts-rfc)対応の新たなRouterを利用しているので現在非対応です。
:::

これだけでNext.jsの世界で履歴ごとの状態復元が可能になります。

以下は実際にrecoil-sync-nextを利用する前後の挙動の違いです。どちらも同じようにrecoilのatomで状態管理しているため、利用前は状態をページ間で共有していますが、recoil-sync-next利用時は**ページごとに状態が分けられている**ことがわかります。

:::details default
![](/images/recoil-sync-next/default.gif)
:::

:::details recoil-sync-next demo
![](/images/recoil-sync-next/demo.gif)
:::

recoil-sync-nextを使う場合、

- 復元されて欲しい状態 -> recoil(syncEffect)
- 復元されて欲しくない状態 -> useState、recoil(syncEffectなし)

のような使い分けになっていくかと思ったのですが、実際に使ってみると「復元されてほしくない状態」というのがほとんど見当たりませんでした。もちろん画面仕様にもよりますが、大抵のUIパーツはやはり復元されて欲しいものなので、現在導入しているプロジェクトでは多用することとなりました。

## 応用編

### react-hook-formとの連携

昨今だとNext.jsでFormを作るのに[react-hook-form](https://react-hook-form.com/)を利用している方も多いのではないでしょうか？以下にreact-hook-formとrecoil-sync-nextで復元可能なFormのサンプルを実装しています。

https://github.com/recruit-tech/recoil-sync-next/blob/main/examples/react-hook-form

長くなってしまうので詳細な説明は省きますが、`useFormSync`というカスタムhooksを定義して、以下のように利用することでreact-hook-formとrecoil-sync-nextの連携を実現しています。

```ts
type FormState = {
  name: string
}

// import { initializableAtom } from 'recoil-sync-next'
// defaultなしでatomを作成するutility
const formState = initializableAtom<FormState>({
  key: 'formState',
  effects: [
    syncEffect({
      refine: object({
        name: string(),
      }),
    }),
  ],
})

// config: useForm(react-hook-form)のdefaultValues以外の引数
const { /* ... */ } = useFormSync<FormState>(formState({ name: 'sato' }), config)
```

### URL Persistenceの注意点

URL Persistenceの利用目的は基本的に共有可能・ブックマーク可能なURLを生成することにあります。利用例としては、検索条件をURLに含む場合など挙げられます。

注意点として、Formの内容をURL Persistenceで**GETパラメータに保存するのは危険です**。ユーザー環境で履歴やリクエストがロギングされている場合、**個人情報が流出する可能性**があります。

### Next.jsのリロード時

[先述の通り](#:~:text=実はリロード対応できてない)、Next.jsの履歴のkeyはリロード時に破棄されます。

### Next.jsのapp directory

こちらも[先述の通り](#:~:text=app%20directory)、2022/10段階ではまだbetaのapp directoryには未対応です。

## まとめ

冒頭でも述べた通り、この手の「戻る」の体験はユーザーのストレスに繋がり、快適なWebブラウジングを阻害します。しかし、recoil-syncやrecoil-sync-nextを利用することで簡単にこれらの体験は改善します。もっと多くの人に使って欲しいし、一方でNext.jsを使っていない人もこの「戻る」の体験を重視して、より良い体験のサイトが増えるといいなと思います。本稿を通じてこの議論が少しでも増えたら嬉しいです。
