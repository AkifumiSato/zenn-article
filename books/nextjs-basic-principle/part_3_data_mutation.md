---
title: "データ操作とServer Actions"
---

## 要約

データ操作はServer Actionsで実装することを基本としましょう。

## 背景

Pages Routerではデータ取得のために[`getServerSideProps()`↗︎](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props)や[`getStaticProps()`↗︎](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-static-props)が提供されてましたが、データ操作アプローチは公式には提供されていませんでした。そのため、クライアントサイドを主体、または[API Routes↗︎](https://nextjs.org/docs/pages/building-your-application/routing/api-routes)を統合した3rd partyライブラリによるデータ操作の実装パターンが多く存在します。

- [SWR↗︎](https://swr.vercel.app/)
- [React Query↗︎](https://react-query.tanstack.com/)
- GraphQL
  - [Apollo Client↗︎](https://www.apollographql.com/docs/react/)
  - [Relay↗︎](https://relay.dev/)
- [tRPC↗︎](https://trpc.io/)
- etc...

しかし、API RouteはApp Routerにおいて[Route Handler↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/route)となり、定義の方法や参照できる情報などが変更されました。また、App Routerは多層のキャッシュを活用しているため、データ操作時にはキャッシュのrevalidate機能との統合が必要となるため、上記にあげたライブラリや実装パターンをApp Routerで利用するには多くの工夫や実装が必要となります。

## 設計・プラクティス

App Routerでのデータ操作は、従来からある実装パターンではなく[Server Actions↗︎](https://nextjs.org/docs/app/getting-started/updating-data)^[データ操作を伴うServer Functionsは、**Server Actions**と呼ばれます。[参考↗︎](https://nextjs.org/docs/app/getting-started/updating-data#what-are-server-functions)]を利用することが推奨されています。これにより、tRPCなどの3rd partyライブラリなどなしにクライアント・サーバーの境界を超えて関数を呼び出すことができ、データ変更処理を容易に実装できます。

:::message
Server Actionsはクライアント・サーバーの境界を超えて関数を呼び出しているように見えますが、実際には当然通信処理が伴うため、引数や戻り値には[Reactがserialize可能なもの↗︎](https://ja.react.dev/reference/rsc/use-server#serializable-parameters-and-return-values)のみを利用できます。

詳しくは[クライアントとサーバーのバンドル境界](part_2_bundle_boundary)を参照ください。
:::

```tsx :app/create-todo.ts
"use server";

export async function createTodo(formData: FormData) {
  // ...
}
```

```tsx :app/page.tsx
"use client";

import { createTodo } from "./create-todo";

export default function CreateTodo() {
  return (
    <form action={createTodo}>
      {/* ... */}
      <button>Create Todo</button>
    </form>
  );
}
```

上記の実装例では、サーバーサイドで実行される関数`createTodo()`をClient Components内の`<form action={createTodo}>`で渡しているのがわかります。このformを実際にsubmitすると、サーバーサイドで`createTodo()`が実行されます。

このように、非常にシンプルな実装でクライアントサイドからサーバーサイド関数を呼び出せることで、開発者はデータ操作の実装に集中できます。Server ActionsはReactの仕様ですが、実装はフレームワークに統合されているので、他にも以下のようなNext.jsならではのメリットが得られます。

### キャッシュのrevalidate

Next.jsは多層のキャッシュを活用しているため、データ操作時には関連するキャッシュのrevalidateが必要になります。Server Actions内で[`revalidatePath()`↗︎](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)や[`revalidateTag()`↗︎](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)を呼び出すと、サーバーサイドの関連するキャッシュ([Data Cache↗︎](https://nextjs.org/docs/app/guides/caching#data-cache)や[Full Route Cache↗︎](https://nextjs.org/docs/app/guides/caching#full-route-cache))とクライアントサイドのキャッシュ([Router Cache↗︎](https://nextjs.org/docs/app/guides/caching#client-side-router-cache))がrevalidateされます。

```tsx :app/actions.ts
"use server";

export async function updateTodo() {
  // ...
  revalidateTag("todos");
}
```

:::message alert
Server Actionsで`revalidatePath()`/`revalidateTag()`もしくは`cookies.set()`/`cookies.delete()`を呼び出すと、Router Cacheが**全て**破棄され、呼び出したページのServer Componentsが再レンダリングされます。
必要以上に多用するとパフォーマンス劣化の原因になるので注意しましょう。
:::

### redirect時の通信効率

Next.jsではサーバーサイドで呼び出せる[`redirect()`↗︎](https://nextjs.org/docs/app/api-reference/functions/redirect)という関数があります。データ操作後にページをリダレイクトしたいことはよくあるユースケースですが、`redirect()`をServer Actions内で呼び出すとレスポンスにリダイレクト先ページの[RSC Payload↗︎](https://nextjs.org/docs/app/getting-started/server-and-client-components#on-the-server)が含まれるため、HTTPリダイレクトをせずに画面遷移できます。これにより、従来データ操作リクエストとリダイレクト後ページ情報のリクエストで2往復は必要だったhttp通信が、1度で済みます。

```tsx :app/actions.ts
"use server";

import { redirect } from "next/navigation";

export async function createTodo(formData: FormData) {
  console.log("create todo: ", formData.get("title"));

  redirect("/thanks");
}
```

上記のServer Actionsを実際に呼び出すと、遷移先の`/thanks`のRSC Payloadが含まれたレスポンスが返却されます。

```text
2:I[3099,[],""]
3:I[2506,[],""]
0:["lxbJ3SDwnGEl3RnM3bOJ4",[[["",{"children":["thanks",{"children":["__PAGE__",{}]}]},"$undefined","$undefined",true],["",{"children":["thanks",{"children":["__PAGE__",{},[["$L1",[["$","h1",null,{"children":"Thanks page."}],["$","p",null,{"children":"Thank you for submitting!"}]]],null],null]},["$","$L2",null,{"parallelRouterKey":"children","segmentPath":["children","thanks","children"],"error":"$undefined","errorStyles":"$undefined","errorScripts":"$undefined","template":["$","$L3",null,{}],"templateStyles":"$undefined","templateScripts":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","styles":null}],null]},[["$","html",null,{"lang":"en","children":["$","body",null,{"children":["$","$L2",null,{"parallelRouterKey":"children","segmentPath":["children"],"error":"$undefined","errorStyles":"$undefined","errorScripts":"$undefined","template":["$","$L3",null,{}],"templateStyles":"$undefined","templateScripts":"$undefined","notFound":[["$","title",null,{"children":"404: This page could not be found."}],["$","div",null,{"style":{"fontFamily":"system-ui,\"Segoe UI\",Roboto,Helvetica,Arial,sans-serif,\"Apple Color Emoji\",\"Segoe UI Emoji\"","height":"100vh","textAlign":"center","display":"flex","flexDirection":"column","alignItems":"center","justifyContent":"center"},"children":["$","div",null,{"children":[["$","style",null,{"dangerouslySetInnerHTML":{"__html":"body{color:#000;background:#fff;margin:0}.next-error-h1{border-right:1px solid rgba(0,0,0,.3)}@media (prefers-color-scheme:dark){body{color:#fff;background:#000}.next-error-h1{border-right:1px solid rgba(255,255,255,.3)}}"}}],["$","h1",null,{"className":"next-error-h1","style":{"display":"inline-block","margin":"0 20px 0 0","padding":"0 23px 0 0","fontSize":24,"fontWeight":500,"verticalAlign":"top","lineHeight":"49px"},"children":"404"}],["$","div",null,{"style":{"display":"inline-block"},"children":["$","h2",null,{"style":{"fontSize":14,"fontWeight":400,"lineHeight":"49px","margin":0},"children":"This page could not be found."}]}]]}]}]],"notFoundStyles":[],"styles":null}]}]}],null],null],[null,"$L4"]]]]
4:[["$","meta","0",{"name":"viewport","content":"width=device-width, initial-scale=1"}],["$","meta","1",{"charSet":"utf-8"}]]
1:null
```

### JavaScript非動作時・未ロード時サポート

Next.jsのServer Actionsでは`<form>`の`action`propsにServer Actionsを渡すと、ユーザーがJavaScriptをOFFにしてたり、JavaScriptファイルが未ロードであっても動作します。

:::message
[公式ドキュメント↗︎](https://nextjs.org/docs/app/getting-started/updating-data#server-components)では「Progressive Enhancementのサポート」と記載されていますが、厳密にはJavaScript非動作環境のサポートとProgressive Enhancementは異なると筆者は理解しています。詳しくは以下をご参照ください。

https://developer.mozilla.org/ja/docs/Glossary/Progressive_Enhancement

:::

これにより、[FID↗︎](https://web.dev/articles/fid?hl=ja)(First Input Delay)の向上も見込めます。実際のアプリケーション開発においては、Formライブラリを利用しつつServer Actionsを利用するケースが多いと思われるので、筆者はJavaScript非動作時もサポートしてるFormライブラリの[Conform↗︎](https://conform.guide/)をおすすめします。

https://zenn.dev/akfm/articles/server-actions-with-conform

## トレードオフ

### サイト外で発生するデータ操作

Server Actionsは基本的にサイト内でのみ利用することが可能ですが、データ操作がサイト内でのみ発生するとは限りません。具体的にはヘッドレスCMSでのデータ更新など、サイト外でデータ操作が発生した場合にも、Next.jsで保持しているキャッシュをrevalidateする必要があります。

Route Handlerが`revalidatePath()`などを扱えるのはまさに上記のようなユースケースをフォローするためです。サイト外でデータ操作が行われた時には、Route Handlerで定義したAPIをWeb hookで呼び出すなどしてキャッシュをrevalidateしましょう。

:::message
Router Cacheはユーザー端末のインメモリに保存されており、全ユーザーのRouter Cacheを一括で破棄する方法はありません。上記の方法で破棄できるのは、サーバー側キャッシュのData CacheとFull Route Cacheのみです。
:::

### ブラウザバックにおけるスクロール位置の喪失

Next.jsにおけるブラウザバックではRouter Cacheが利用されます。この際には画面は即時に描画され、スクロール位置も正しく復元されます。

しかし、Server Actionsで`revalidatePath()`などを呼び出すなどすると、Router Cacheが破棄されます。Router Cacheがない状態でブラウザバックを行うと即座に画面を更新できないため、スクロール位置がうまく復元されないことがあります。

### Server Actionsの呼び出しは直列化される

Server Actionsは直列に実行されるよう設計されているため、同時に実行できるのは一つだけとなります。

https://quramy.medium.com/server-actions-%E3%81%AE%E5%90%8C%E6%99%82%E5%AE%9F%E8%A1%8C%E5%88%B6%E5%BE%A1%E3%81%A8%E7%94%BB%E9%9D%A2%E3%81%AE%E7%8A%B6%E6%85%8B%E6%9B%B4%E6%96%B0-35acf5d825ca

本書や公式ドキュメントで扱ってるような利用想定の限りではこれが問題になることは少ないと考えられますが、Server Actionsを高頻度で呼び出すような実装では問題になることがあるかもしれません。

そういった場合は、そもそも高頻度にServer Actionsを呼び出すような設計・実装を見直しましょう。
