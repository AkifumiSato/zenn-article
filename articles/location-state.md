---
title: "ブラウザバックで壊れないstate管理を実現する`location-state`"
emoji: "⏳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "nextjs"]
published: true
---

この記事は本日リリースした[location-state](https://github.com/recruit-tech/location-state)というライブラリの紹介記事です。

## モチベーション

Reactのstate管理は、様々な分類が可能です。筆者が過去に書いた記事「[スコープとライフタイムで考えるReact State再考](https://zenn.dev/terrierscript/articles/react-state-scope)」では、stateの分類は大きく以下2つの観点で分類ができると述べました。

- **スコープによる分類**
- **ライフタイム（=stateの生存期間）による分類**

詳しく知りたい方はこの記事を読んでいただきたいのですが、今でもstate管理というと多くの場合スコープによる分類の話が多く、ライフタイムによる分類の話はあまり聞かない気がします。

### なぜライフタイム観点が重要か

ライフタイムを意識せずに実装した場合に発生するのが、遷移時に状態が破棄され復元されない、つまり**ブラウザバックでstateが壊れる**という問題です。この問題については以下の記事で、Vercelの社長が2014年にはすでに「historyを壊すべきじゃない」と提唱しています。

https://yosuke-furukawa.hatenablog.com/entry/2014/11/14/141415

残念ながら現在でもこの問題について対応されているサイトは少ないのが実情で、時折ユーザー視点でも[問題提起](https://rentwi.hyuki.net/?1576010373357965312)がされています。MPAの場合、ブラウザによって[Domやデータの復元が行われる](https://zenn.dev/akfm/articles/recoi-sync-next#%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%83%90%E3%83%83%E3%82%AF%E6%99%82%E3%81%AEui%E7%8A%B6%E6%85%8B%E3%81%AE%E5%BE%A9%E5%85%83)ことがあるため、この問題は特にSPAにおいて顕著です。

この問題を解決すべく作られたのが[location-state](https://github.com/recruit-tech/location-state)です。

### recoil-sync-nextとの棲み分け

余談ですが、実はこの問題に対する取り組みとして筆者は過去に[recoil-sync-next](https://github.com/recruit-tech/recoil-sync-next)というライブラリ開発にも携わらせていただきました。このことについても、「[Next.jsで戻る厨を満たすrecoil-sync-next](https://zenn.dev/akfm/articles/recoi-sync-next)」という記事で紹介させていただきました。

上記執筆後、[recoil](https://recoiljs.org/)の一部APIとNext.jsを併用した時にメモリリークが発生しうるこがわかりました。recoil開発チームは当時からメモリリーク系の問題を抱えていることは認識しており、対策としてガベージコレクション機能の実装が実験的に行われていたため、筆者は将来的なアップデートで解決するだろうと考えていました。しかしその後、Metaのレイオフの影響なのかrecoil自体のメンテが滞り気味になってしまい、待たれていたガベージコレクションの実装が進捗しない=解決が待たれていたこのメモリリーク問題も解消されず、recoil・recoil-syncに依存した実装のままだと厳しいという気持ちが生まれたことで、本ライブラリの開発に至りました。

:::message
[recoilのコミット数のグラフ](https://github.com/facebookexperimental/Recoil/graphs/contributors)をみると明らかに2023年に入った頃から以前ほど活発にメンテンナンスされていないことがわかります。
最もコミットしていた[drarmstr](https://github.com/drarmstr)も[今年の4月を最後](https://github.com/facebookexperimental/Recoil/commits?author=drarmstr)にコミットしていません。
:::

## location-state

閑話休題。話は戻り、location-stateの紹介です。

[location-state](https://github.com/recruit-tech/location-state)は履歴位置に同期してstateを管理することができるライブラリです。現時点ではNext.jsをメインにテスト・サポートしており、[App Router](https://nextjs.org/docs/app)と[Pages Router](https://nextjs.org/docs/pages)の両方で利用できます。location-stateはScopedパッケージで開発されており、以下のようなパッケージ構成になっています。

- `@location-state/core`: location-stateのコア機能を提供するパッケージ
- `@location-state/next`: Next.jsのPages Routerで利用するためのパッケージ

`@location-state/core`は[Navigation Api](https://github.com/WICG/navigation-api)で同期させることができるため、基本的にフレームワークを選ばず単体利用することが可能です。一方で`@location-state/next`はPages Router専用のパッケージで、Navigation Apiではなく[router.events](https://nextjs.org/docs/pages/api-reference/functions/use-router#routerevents)を利用して同期しています。詳細については後述します。

要望が多ければ他フレームワークの専用パッケージも開発するかもしれませんが、当面はNext.jsを中心にサポートする予定です。

## Next.js App Routerでの使い方

Next.js App Routerで利用するには`@location-state/core`のみOKです。

```
npm install @location-state/core
# or
yarn add @location-state/core
# or
pnpm add @location-state/core
```

`@location-state/core`は`useLocationState`をはじめいくつかのhooksを提供しています。これを利用することで、履歴位置に同期してstateを管理することができます。

このhooksを利用するには、`LocationStateProvider`を上位のコンポーネントで呼び出す必要があります。以下は`layout.tsx`をServer Componentのままにして、`Providers.tsx`にClient Componentを切り出した例です。

```tsx
// src/app/Providers.tsx
"use client";

import { LocationStateProvider } from "@location-state/core";

export function Providers({ children }: { children: React.ReactNode }) {
  return <LocationStateProvider>{children}</LocationStateProvider>;
}
```

```tsx
// src/app/layout.tsx
import { Providers } from "./Providers";

// ...snip...

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

あとは利用したい箇所で`useLocationState`を呼び出すだけです。

```tsx
// src/components/Counter.tsx
"use client";

import { useLocationState } from "@location-state/core";

export function Counter() {
  const [counter, setCounter] = useLocationState({
    name: "counter",
    defaultValue: 0,
    storeName: "session",
  });

  return (
    <div>
      <p>
        storeName: <b>{storeName}</b>, counter: <b>{counter}</b>
      </p>
      <button onClick={() => setCounter(counter + 1)}>increment</button>
    </div>
  );
}
```

現時点で`useLocationState`の引数で渡せるオプションは以下の4つです。

- `name`: stateを一意に判別する名前
- `defaultValue`: stateの初期値
- `storeName`: stateの保存先。`session`と`url`の2つが利用可能（カスタマイズ可能）
- `refine?`: state復元時のバリデーション関数。`undefined`を返すと初期値となる

### @location-state/coreの注意点

App Routerでは、履歴を一意に特定するkeyは公開されていません。これについては過去の記事でも何度か触れていますが、Next.jsに提案してみたものの進展がない状況です。

https://github.com/vercel/next.js/discussions/47242

そのため`@location-state/core`はデフォルトでは[Navigation Api](https://github.com/WICG/navigation-api)を利用することで履歴を一意に特定する方法をとっています。しかしこのNavigation APIも[SafariやFirefoxでまだ実装されていません](https://caniuse.com/?search=navigation%20api)。

そこで、location-stateではChrome以外でも動作するように、`@location-state/core/unsafe-navigation`というAPIを提供しています。

```tsx
// src/app/Providers.tsx
"use client";

import { LocationStateProvider, NavigationSyncer } from "@location-state/core";
import { unsafeNavigation } from "@location-state/core/unsafe-navigation";

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <LocationStateProvider syncer={new NavigationSyncer(unsafeNavigation)}>
      {children}
    </LocationStateProvider>
  );
}
```

これはNavigation APIの挙動を部分的にサポートしたpolyfill的なものですが、ライブラリとして積極的なテスト・サポートをしているわけではありません。一応自身のSafariやFirefoxで動作することは確認していますが、利用する方は各々でサポートしたいブラウザ環境でのテストを実施することをお勧めします。

## Next.js Pages Routerでの使い方

一方でPages Routerの場合、`@location-state/core/unsafe-navigation`の代わりに`@location-state/next`を利用すれば、Navigation APIに依存せずに`useLocationState`を利用できます。

```
npm install @location-state/core @location-state/next
# or
yarn add @location-state/core @location-state/next
# or
pnpm add @location-state/core @location-state/next
```

Pages Routerで利用する際の実装上の違いは、Providerの設定が違うのみです。`@location-state/next`から`useNextPagesSyncer`で**syncer**（本ライブラリにおいて遷移周りのハンドリングを担うもの）を取得し、Providerに渡しています。

```tsx
// src/pages/_app.tsx
import { LocationStateProvider } from "@location-state/core";
import { useNextPagesSyncer } from "@location-state/next";
import type { AppProps } from "next/app";

export default function MyApp({ Component, pageProps }: AppProps) {
  const syncer = useNextPagesSyncer();
  return (
    <LocationStateProvider syncer={syncer}>
      <Component {...pageProps} />
    </LocationStateProvider>
  );
}
```

hooksについてはApp Routerで利用する際と違いはありません。

## location-stateの今後

まだ一部ドキュメントが不足しているので拡充していこうと思っています。またよくある実装ユースケースとして、[react-hook-form](https://react-hook-form.com/)との併用が考えられます。こちらについても簡単に統合できるようなパッケージの開発を進めていこうと考えています。

## 感想

今回Scopedパッケージだったこともあり、monorepoでのライブラリ開発の知見も多く得られました。この辺りは後日また記事にまとめてみようかと思っています。

また、今回はrecoil-sync-nextと比較するとrecoilやrecoil-syncのような依存がない分、**軽量かつより採用しやすいライブラリになったんじゃないか**と思っています。

この記事を通してSPA遷移時に状態が破棄され復元されない問題への問題意識が高まり、また、その解決に本ライブラリが役に立てばとても嬉しく思います。
