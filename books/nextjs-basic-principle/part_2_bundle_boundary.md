---
title: "クライアントとサーバーのバンドル境界"
---

## 要約

`"use client"`や`"use server"`は実行環境を示すものではありません。これらはバンドラに**バンドル境界**を宣言するためのものです。

Server Bundleでのみ利用可能なモジュールは、`server-only`を使ってモジュールを保護しましょう。

:::message
本章の解説内容は、[Dan Abramov氏の記事↗︎](https://overreacted.io/what-does-use-client-do/)を参考にしています。より詳細に知りたい方は元記事をご参照ください。
:::

## 背景

RSCは多段階計算^[参考: [一言で理解するReact Server Components↗︎](https://zenn.dev/uhyo/articles/react-server-components-multi-stage)]アーキテクチャであり、サーバー側処理とクライアント側処理の2段階計算で構成されます。これはつまり、バンドルもサーバーバンドル（**Server Bundle**）とクライアントバンドル（**Client Bundle**^[Next.jsの内部的には、Client Bundleはブラウザで実行されるCSR用とNode.jsで実行されるSSR/SSG用それぞれのバンドルファイルが作成されるようです]）の2つに分けられることを意味し、Server ComponentsやServer FunctionsはServer Bundleに、Client ComponentsはClient Bundleに含められます。

多くの人は、`"use client"`や`"use server"`などのディレクティブがバンドルに関する重要なルールであることは知っています。しかし、これらのディレクティブの役割については「実行環境を示すためのもの」と誤解されることがよくあるようです^[`"use server"`に関する誤解を発端とした議論例: [Dan Abramov氏のBlueSkyでのやりとり↗︎](https://bsky.app/profile/danabra.mov/post/3lnw334g5jc24)]。実際には、これらのディレクティブは**実行環境を示すものではありません**。

## 設計・プラクティス

`"use client"`や`"use server"`は、**バンドル境界**を宣言するためのものです。

- `"use client"`:
  - **サーバー -> クライアント**のバンドル境界（**Client Boundary**）
  - サーバーへClient Componentsを公開する
- `"use server"`:
  - **クライアント -> サーバー**のバンドル境界（**Server Boundary**）
  - クライアントへServer Functionsを公開する

これにより、RSCでは2つのバンドルを1つのプログラムとして表現することができます。`"use client"`や`"use server"`は、RSCにおいて最も重要な責務を負ったルールです。これらの役割を正しく理解することは、Next.jsにおいても非常に重要です。

:::message alert

以下はよくある誤解です。

##### Q. Server Componentsには`"use server"`を付ける必要がある？

いいえ、前述の通り`"use server"`はServer Boundaryを宣言するためのものであり、Server Componentsを定義するためのものではありません。

##### Q. ではServer Componentsを定義するにはどうすればいい？

Next.jsでは、デフォルトでServer Componentsとなるので何も指定する必要はありません。

##### Q. Client ComponentsにServer Componentsを含めるにはどうすればいい？

Client Bundle内でServer Componentsを`import`することはできません。ただし、Client Componentsの`children`などにServer Componentsを渡すことは可能です。これは[Compositionパターン](part_2_composition_pattern)と呼ばれます。

:::

:::message
RSCにおけるバンドラの役割については、[uhyoさんの資料↗︎](https://speakerdeck.com/uhyo/rscnoshi-dai-nireacttohuremuwakunojing-jie-wotan-ru?slide=16)で詳しく解説されています。興味のある方はこちらをご参照ください。
:::

### モジュールの依存関係とバンドル境界

例として、ユーザー情報の編集ページで考えてみましょう。このページの`page.tsx`は、以下のような依存関係で構成されていると仮定します。

```
page.tsx
├── user-fetcher.ts
└── user-profile-form.tsx
    └── submit-button.tsx
```

以下はこれらのファイルの実装イメージです。

:::details 各ファイルの実装イメージ

```tsx:page.tsx
import { getUser } from "./user-fetcher";
import { UserProfileForm } from "./user-profile-form";

export default async function Page() {
  const user = await getUser();

  return (
    <div>
      <h1>User Profile</h1>
      <UserProfileForm defaultUser={user} />
    </div>
  );
}
```

```tsx:user-profile-form.tsx
"use client";

import { SubmitButton } from "./submit-button";
// ...省略...

export function UserProfileForm({ defaultUser }: { defaultUser: UserProfile }) {
  // ...省略...
  const onClick = () => {
    // ...省略...
  };

  return (
    <form>
      {/* ...省略... */}
      <SubmitButton onClick={onClick} />
    </form>
  );
}
```

```tsx:submit-button.tsx
export function SubmitButton({ onClick }: { onClick: () => void }) {
  return <button onClick={onClick}>submit</button>;
}
```

```ts:user-fetcher.ts
export async function getUser() {
  // ...省略...
}
```

:::

これらのモジュールの依存関係をツリーで図示すると、以下のようになります。

![RSCのバンドル境界](/images/nextjs-basic-principle/rsc-bundle-boundary-1.png)

`submit-button.tsx`は`"use client"`を含みませんが、`"use client"`を含む`user-profile-form.tsx`から`import`されているため、Client Bundleに含まれます。このように、モジュールの依存関係にはバンドルの境界が存在し、`"use client"`はClient Boundaryを担います。

ここにさらに、フォームのサブミット時に呼び出されるServer Functionsを含む`update-profile-action.ts`を追加すると、以下のようになります。

:::details `update-profile-action.ts`の実装イメージ

```diff:user-profile-form.tsx
 // ...省略...
+ import { updateProfile } from "./update-profile-action";

 export function UserProfileForm({ defaultUser }: { defaultUser: UserProfile }) {
   // ...省略...
-  const onClick = () => {
+  const onClick = async () => {
+      await updateProfile(formData);
     // ...省略...
   };

   // ...省略...
 }
```

```ts:update-profile-action.ts
export async function updateProfile(formData: FormData) {
  "use server";

  // ...省略...
}
```

:::

![RSCのバンドル境界](/images/nextjs-basic-principle/rsc-bundle-boundary-2.png)

このように、`"use server"`はServer Boundaryを担います。

:::message
**Client Boundaryとなる**Client ComponentsはBundleを跨ぐため、受け取るPropsは原則として[Reactがserialize可能なもの↗︎](https://ja.react.dev/reference/rsc/use-client#serializable-types)である必要があります。一方**Client Boundaryでない**Client Componentsが受け取れるPropsは特に制約がありません。
:::

### 「2つの世界、2つのドア」

Dan Abramov氏は[前述の記事↗︎](https://overreacted.io/what-does-use-client-do/#two-worlds-two-doors)にて、`"use client"`と`"use server"`の役割を「2つの世界、2つのドア」という言葉で説明しています。RSCにはServer BundleとClient Bundleという2つの世界があり、これらの世界を開くドアが`"use client"`と`"use server"`です。

![境界とディレクティブ](/images/nextjs-basic-principle/rsc-layer.png)

## トレードオフ

### ファイル単位の`"use server"`による予期せぬエンドポイントの公開

`"use server"`は関数単位でもファイル単位でも宣言が可能です。ファイル単位で`"use server"`を宣言した場合、`export`された全ての関数はServer Functionsとして扱われます。これにより、意図せず関数がエンドポイントとして公開される可能性があるので、注意しましょう。

詳しくは以下の記事で解説されているので、ご参照ください。

https://zenn.dev/moozaru/articles/b0ef001e20baaf

### `server-only`

Server Bundleでのみ利用可能なモジュールを実装することは、よくあるユースケースです。このような場合には[`server-only`↗︎](https://www.npmjs.com/package/server-only)を使うことで、モジュールがServer Bundleでのみ利用されるよう保護できます。

```tsx
import "server-only";
```

もしClient Bundle内で`import "server-only"`を含むモジュールが見つかった場合、Next.jsはビルドできずエラーとなります。

逆に、Client Bundleでのみ利用可能なモジュールを実装する場合には、[`client-only`↗︎](https://www.npmjs.com/package/client-only)を利用することでClient Bundleでのみ利用されるよう保護できます。
