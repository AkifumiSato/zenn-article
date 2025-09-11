---
title: "クライアントとサーバーのバンドル境界"
---

## 要約

`"use server"`や`"use client"`は実行環境を示すものではありません。これらはバンドラに**バンドル境界**を宣言するためのものです。

サーバーバンドルでのみ利用可能なモジュールを作成する場合は、`server-only`を使ってモジュールを保護しましょう。

:::message
本章の解説内容は、[Dan Abramov氏の記事](https://overreacted.io/what-does-use-client-do/)を参考にしています。より詳細に知りたい方は元記事をご参照ください。
:::

## 背景

RSCは多段階計算^[参考: [一言で理解するReact Server Components](https://zenn.dev/uhyo/articles/react-server-components-multi-stage)]アーキテクチャであり、サーバー側処理とクライアント側処理の2段階計算で構成されます。これはつまり、バンドルも**サーバーバンドル**と**クライアントバンドル**の2つに分けられることを意味します。

多くの人は、`"use server"`や`"use client"`などのディレクティブがバンドルに関する重要なルールであることは知っています。しかし、これらのディレクティブの役割については「実行環境を示すためのもの」と誤解されることがよくあるようです^[`"use server"`に関する誤解を発端とした議論例: [Dan Abramov氏のBlueSkyでのやりとり](https://bsky.app/profile/danabra.mov/post/3lnw334g5jc24)]。実際には、これらのディレクティブは**実行環境を示すものではありません**。

## 設計・プラクティス

`"use server"`や`"use client"`は、**バンドル境界**を宣言するためのものです。

- `"use server"`: **サーバーバンドルの境界**を宣言
- `"use client"`: **クライアントバンドルの境界**を宣言

これにより、RSCでは2つのバンドルを1つのプログラムとして表現することができます。`"use server"`や`"use client"`は、RSCにおいて最も重要な責務を負ったルールです。これらの役割を正しく理解することは、Next.jsにおいても非常に重要です。

:::message alert

以下はよくある誤解です。

##### Q. Server Componentsには`"use server"`を付ける必要がある？

いいえ、前述の通り`"use server"`はサーバーバンドルの境界を宣言するためのものであり、Server Componentsを定義するためのものではありません。

##### Q. ではServer Componentsを定義するにはどうすればいい？

Next.jsでは、デフォルトでServer Componentsとなるので何も指定する必要はありません。

:::

:::message
RSCにおけるバンドラの役割については、[uhyoさんの資料](https://speakerdeck.com/uhyo/rscnoshi-dai-nireacttohuremuwakunojing-jie-wotan-ru?slide=16)で詳しく解説されています。興味のある方はこちらをご参照ください。
:::

### モジュールの依存関係とバンドル境界

例として、ユーザー情報の編集ページで考えてみましょう。このページの`page.tsx`は、以下のような依存関係で構成されていると仮定します。

```
page.tsx
├── user-fetcher.ts
└── user-profile-form.tsx
    └── user-profile-schema.ts
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
      <UserProfileForm defaultValue={user} />
    </div>
  );
}
```

```tsx:user-profile-form.tsx
"use client";

import { UserProfile } from "./user-profile-schema";
import { useForm } from "@conform-to/react";
import { zodResolver } from "@hookform/resolvers/zod";

export function UserProfileForm({
  defaultValue,
}: {
  defaultValue: UserProfile;
}) {
  const { register, handleSubmit } = useForm({
    resolver: zodResolver(UserProfile),
    defaultValue,
  });

  // ...formの組み立て...
}
```

```tsx:user-profile-schema.ts
import { z } from "zod";

export const UserProfile = z.object({
  name: z.string(),
  age: z.number(),
});

export type UserProfile = z.infer<typeof UserProfile>;
```

```ts:user-fetcher.ts
export async function getUser() {
  // ...APIよりユーザー情報を取得...
}
```

:::

これらのモジュールの依存関係をツリーで図示すると、以下のようになります。

![RSCのバンドル境界](/images/nextjs-basic-principle/rsc-bundle-boundary-1.png)

`user-profile-schema.ts`は`"use client"`を含みませんが、`"use client"`を含む`user-profile-form.tsx`から`import`されているため、クライアントバンドルに含まれます。このように、モジュールの依存関係にはバンドルの境界が存在し、`"use client"`はサーバーバンドルとクライアントバンドルの境界を担います。

ここにさらに、フォームのサブミット時に呼び出されるServer Functionsを含む`update-profile-action.ts`を追加すると、以下のようになります。

:::details `update-profile-action.ts`の実装イメージ

```ts:update-profile-action.ts
export async function updateProfile(formData: FormData) {
  "use server";

  // ...ユーザー情報の更新をAPIリクエスト...
}
```

:::

![RSCのバンドル境界](/images/nextjs-basic-principle/rsc-bundle-boundary-2.png)

このように、`"use server"`はクライアントバンドルとサーバーバンドルの境界を担います。

### 「2つの世界、2つのドア」

Dan Abramov氏は[前述の記事](https://overreacted.io/what-does-use-client-do/#two-worlds-two-doors)にて、`"use client"`と`"use server"`の役割を「2つの世界、2つのドア」という言葉で説明しています。RSCにはサーバーバンドルとクライアントバンドルという2つの世界があり、これらの世界を開くドアが`"use client"`と`"use server"`です。

![境界とディレクティブ](/images/nextjs-basic-principle/rsc-layer.png)

## トレードオフ

### `server-only`

サーバーバンドルでのみ利用可能なモジュールを実装することは、よくあるユースケースです。このような場合には[`server-only`](https://www.npmjs.com/package/server-only)を使うことで、モジュールがサーバーバンドルでのみ利用されるよう保護できます。

```shell
$ pnpm add server-only
```

```tsx
import "server-only";
```

もしクライアントバンドル内で`import "server-only";`を含むモジュールが見つかった場合、Next.jsはビルドできずエラーとなります。

逆に、クライアントバンドルでのみ利用可能なモジュールを実装する場合には、[`client-only`](https://www.npmjs.com/package/client-only)を利用することでクライアントバンドルでのみ利用されるよう保護できます。

### ファイル単位の`"use server"`による予期せぬエンドポイントの公開

`"use server";`は関数単位でもファイル単位でも宣言が可能です。ファイル単位で`"use server";`を宣言した場合、`export`された全ての関数はServer Functionsとして扱われます。これにより、意図せず関数がエンドポイントとして公開される可能性があるので、注意しましょう。

詳しくは以下の記事で解説されているので、ご参照ください。

https://zenn.dev/moozaru/articles/b0ef001e20baaf
