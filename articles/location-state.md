---
title: "遷移と同期するstate管理ライブラリ、`location-state`"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "nextjs"]
published: false
---

この記事は最近リリースした[location-state](https://github.com/recruit-tech/location-state)というライブラリの紹介記事です。

## モチベーション

Reactの状態管理については、様々な分類が可能です。筆者が過去に書いた記事[スコープとライフタイムで考えるReact State再考](https://zenn.dev/terrierscript/articles/react-state-scope)では、stateの分類は大きく**スコープによる分類**と**ライフタイム（stateの生存期間）による分類**が可能であると述べました。詳しくはこの記事を読んでいただきたいのですが、今でもstate管理というと多くの場合スコープによる分類の話が多く、ライフタイムによる分類の話はあまり聞かない気がします。

このライフタイムによるstateの分類の話をあまり聞かないというのは、それだけ多くの開発者がこの観点を重視していないということです。多くの開発者が開発要件に基づいて実装する上で、スコープの問題に直面します。そのためにスコープの問題は非常に重要です。一方でライフタイムの問題も同様に重要なはずですが、こちらは要件を定義する側がSPA・MPAの挙動について詳細に理解できてないと直接開発要件として定義するのは難しいことでしょう。そのため、定義された要件通りに実装する上でライフタイムによるstateの分類に直面する機会はスコープによる分類に比べて減ってしまい、あまり重要視されていないのではないかと筆者は考えています。

### SPA遷移時に状態が破棄され復元されない問題

ライフタイムを意識せずに実装した場合に発生するのが、「SPA遷移時に状態が破棄され復元されない」という問題です。この問題については[【翻訳】リッチなWebアプリケーションのための7つの原則](https://yosuke-furukawa.hatenablog.com/entry/2014/11/14/141415)など、2014年くらいにはすでに提唱されていますが、残念ながら現在でもこの問題について対応されているSPAサイトは少ないのが実情です。

この問題を解決すべく作られたのが[location-state](https://github.com/recruit-tech/location-state)です。

### recoil-sync-nextとの棲み分け

実はこの問題に対する取り組みとして、筆者は過去に[recoil-sync-next](https://github.com/recruit-tech/recoil-sync-next)というライブラリ開発にも微力ながら携わらせていただきました。このことについても、[Next.jsで戻る厨を満たすrecoil-sync-next](https://zenn.dev/akfm/articles/recoi-sync-next)という記事で紹介させていただきました。

上記記事執筆の後分かったことですが、[recoil](https://recoiljs.org/)とNext.jsを併用した時に一部APIを利用することでメモリリークが発生することがあることがわかりました。当時からrecoilのissueでこの問題は認識されており、筆者としては解決を待っていました。

しかしその後、[recoil](https://recoiljs.org/)のメンテが滞り気味になってしまい、解決が待たれていたこのメモリリーク問題も解消されず、recoil・recoil-syncに依存した実装のままだと厳しいという気持ちが生まれ、本ライブラリの開発に至りました。

## location-state

[location-state](https://github.com/recruit-tech/location-state)は履歴位置に同期してstateを管理することができるライブラリです。現時点ではNext.jsをメインにサポートしており、[App Router](https://nextjs.org/docs/app)と[Pages Router](https://nextjs.org/docs/pages)の両方で利用できます。

`location-state`はScopedパッケージで開発されており、以下のようなパッケージ構成になっています。

- `@location-state/core`: `location-state`のコア機能を提供するパッケージ、Next.js App RouterはこちらのみでOK
- `@location-state/next`: Next.jsのPages Routerで利用するためのパッケージ

要望が多ければ他フレームワークのサポートもするかもしれませんが、当面はNext.jsのみサポートする予定です。

## Next.js App Routerでの使い方

前述の通り、Next.js App Routerで利用するには`@location-state/core`のみOKです。

```
npm install @location-state/core
# or
yarn add @location-state/core
# or
pnpm add @location-state/core
```

`@location-state/core`は`useLocationState`をはじめいくつかのhooksを提供しています。これを利用することで、履歴位置に同期してstateを管理することができます。

このhooksを利用するには、`LocationStateProvider`を`layout.tsx`などで呼び出す必要があります。

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
- `storeName`: stateの保存先。`session`と`local`の2つが利用可能（カスタマイズ可能）
- `refine`: state復元時のバリデーション関数。`undefined`を返すと初期値となる

### App Routerでの注意点

App Routerでは、履歴を一意に特定するkeyは公開されていません。これについては過去の記事でも何度か触れていますが、Next.jsに提案してみたものの、現状進展がない状況です。

https://github.com/vercel/next.js/discussions/47242

そのため`location-state`では[Navigation Api](https://github.com/WICG/navigation-api)より履歴を一意に特定する方法をとっています。しかしこのNavigation APIもChrome以外でまだ実装されていません。

そのため他ブラウザをサポートするために、`location-state`では`@location-state/core/unsafe-navigation`というAPIを提供しています。これはNavigation APIの挙動を部分的にサポートしたpolyfill的APIですが、命名通り`unsafe`なので利用する際は注意が必要です。

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

## Next.js Pages Routerでの使い方

一方でPages Routerで使う場合には、`@location-state/next`を利用する必要があります。

```
npm install @location-state/core @location-state/next
# or
yarn add @location-state/core @location-state/next
# or
pnpm add @location-state/core @location-state/next
```

Pages Routerで利用する際の実装上の違いは、Providerの設定が違うのみです。

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

まだ一部ドキュメントが不足しているので拡充していこうと思っています。またよくある実装ユースケースとして、[react-hook-form](https://react-hook-form.com/)との併用が考えられます。こちらについても開発を進めていこうと思っております。

## 感想

今回Scopedパッケージだったこともあり、monorepoでのライブラリ開発の知見も多く得られました。この辺りは後日また記事にまとめてみようかと思っています。

また、今回はrecoil-sync-nextと比較するとrecoilやrecoil-syncのような依存がない分、軽量かつより採用しやすいライブラリになったんじゃないかと思っています。

この記事を通して「SPA遷移時に状態が破棄され復元されない問題」への問題意識が高まり、また、その解決に本ライブラリが役に立てばとても嬉しく思います。
