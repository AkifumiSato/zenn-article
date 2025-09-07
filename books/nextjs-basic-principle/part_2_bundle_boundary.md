---
title: "クライアントとサーバーのバンドル境界"
---

## 要約

`"use client"`や`"use server"`は実行環境を示すものではありません。これらはバンドラに**バンドル境界**を宣言するためのものです。

サーバーバンドルでのみ利用可能なモジュールを作成する場合は、`server-only`を使ってモジュールを保護しましょう。

:::message
本章の解説内容は、[Dan Abramov氏の記事](https://overreacted.io/what-does-use-client-do/)を参考にしています。より詳細に知りたい方は元記事をご参照ください。
:::

## 背景

- RSCではサーバーとクライアント、2つのバンドルが作成される
  - だからバンドラ側の対応が必須
  - RSCとしてのルールが存在する
- RSCのルールである`"use client"`や`"use server"`が「実行環境をマークするためのもの」と誤解されることがよくある
  - これは事実ではない

## 設計・プラクティス

- `"use client"`はクライアントバンドルの境界を、`"use server"`はサーバーバンドルの境界を宣言するためのものである
- RSCでは2つのバンドルを1つのプログラムとして表現することで、以下を実現している
  - **Server ComponentsはClient Componentsを含むことができる**
  - **Client ComponentsはServer Functionsを呼び出すことができる**

![境界とディレクティブ](/images/nextjs-basic-principle/rsc-layer.png)

<!-- TODO: 図を更新 -->

:::message alert

以下はよくある誤解です。

##### Q. Server Componentsには"use server"を付ける必要がある？

いいえ、前述の通り`"use server"`はサーバーバンドルの境界を宣言するためのものです。

##### Q. ではServer Componentsを定義するにはどうすればいい？

Next.jsではデフォルトでServer Componentsなので、何も指定する必要はありません。

:::

### モジュールツリーとバンドル境界

- モジュール依存関係のツリーで境界を表現すると、以下のようになる
  - TODO: [公式の図相当](https://ja.react.dev/reference/rsc/use-client#how-use-client-marks-client-code)を用意
- この境界を宣言するのが、`"use client"`と`"use server"`
- バンドラはこれらを元に、サーバーバンドルとクライアントバンドルどちらに含めるべきか知ることができる
- 前述のDan Abramov氏の言葉を借りれば、「2つの世界、2つのドア」

### アイランドアーキテクチャとの類似性

- アイランドアーキテクチャも同様にサーバーバンドルにクライアントバンドルをOptinできる設計
  - （以前の記事を要約）

## トレードオフ

### `server-only`

- APIキーやサーバー間通信のロジックを含むような、サーバーバンドルでのみ利用可能なモジュールを実装することはよくある
- `server-only`を使うことでモジュールがサーバーバンドルでのみ利用されることを保証できる

```tsx
import "server-only";
```

- 仮にクライアントバンドルに`import "server-only";`を含むモジュールが見つかった場合にはビルドエラーとなる

### ファイル単位の`"use server"`には注意が必要

- ["use server"; でexportした関数が意図せず？公開される](https://zenn.dev/moozaru/articles/b0ef001e20baaf)ことがあるので、ファイル単位で`"use server";`を宣言する際には注意が必要
