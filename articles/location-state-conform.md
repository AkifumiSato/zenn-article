---
title: "@location-state/conformをリリースした"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "conform", "nextjs"]
published: false
---

この記事はlocation-stateをconformに対応させるために開発した、[@location-state/conform](https://www.npmjs.com/package/@location-state/conform)の紹介記事です。

## location-stateとは

location-stateは履歴位置に同期する状態管理ライブラリです。

https://github.com/recruit-tech/location-state

Next.jsなどを採用している場合、ページ内の`useState`は遷移時のunmountで状態が破棄され、ブラウザバック時には**復元されません**。そのため、アコーディオンやform要素の状態はブラウザバック時にはリセットされてしまいます。これはNext.jsに限らず、ReactやVueなどをベースにしたモダンなフロントエンドフレームワークを採用して、クライアントサイドルーティングが発生する場合に起きがちな挙動です。クライアントサイドルーティングが不在なMPAでは、bfcacheやブラウザ側の復元処理によってDOMの状態が復元されます。

筆者もサイト利用時に、formの入力途中で前のページの情報を確認するためにブラウザバックし、再度formに戻ってきたら入力内容が消えていた経験があります。これは、従来のMPAなら復元されていたことでしょう。SPAとMPAでブラウザバック時の挙動が異なることはユーザーにとって望ましくありません。

:::message
ブラウザバック挙動の違いについては、筆者の[過去の記事](https://zenn.dev/akfm/articles/recoi-sync-next#%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%83%90%E3%83%83%E3%82%AF%E6%99%82%E3%81%AEui%E7%8A%B6%E6%85%8B%E3%81%AE%E5%BE%A9%E5%85%83)で詳細に解説しているので、興味がある方はぜひご覧ください。
:::

しかし、開発者が自前で履歴ごとに復元されるような状態管理を実装するのは非常に大変です。これらの課題を解消すべく開発されたのが[@location-state/conform](https://www.npmjs.com/package/@location-state/conform)です。

より詳細にlocation-stateについて知りたい方は、リリース時に書いた以下の記事をご参照ください。

https://zenn.dev/akfm/articles/location-state

## conform

さて、今回はこのlocation-stateがconformに対応したわけなので、conformについても簡単に紹介しておきます。conformは[react-hook-form](https://react-hook-form.com/)などより後発な、Reactのformライブラリです。

https://ja.conform.guide/

主な特徴としては以下が挙げられます。

- validationライブラリとの統合が容易
- 強力なTypeScriptサポート
- Server ActionsやReactのhooksとの親和性が高い
- Progressive Enhancementに対応

筆者はconformを、**Server Actions時代のformライブラリ**として台頭する可能性があると考え、非常に注目しています。以下の記事でより詳細に紹介しているので、興味のある方はぜひご覧ください。

https://zenn.dev/akfm/articles/server-actions-with-conform

## location state conform

ブラウザバック体験を破壊しないようサポートしたいlocation-stateをconformに対応させたのが、今回開発した`@location-state/conform`です。

https://www.npmjs.com/package/@location-state/conform

`@location-state/core`と併用して利用できます。以降は`@location-state/conform`利用前後での挙動の違いや、利用方法について紹介したいと思います。

### @location-state/conformなしでの挙動

まず素のconformの実装と挙動を確認します。location-stateのリポジトリにある[example](https://github.com/recruit-tech/location-state/tree/main/apps/example-next-conform)を簡易化しつつ確認していきたいと思います。

Next.jsでconformを使う時は、`@conform-to/react`と`@conform-to/zod`を利用します。Server Actionsではzod schemaを`parseWithZod`と併用して`submission`を作成し、必要に応じて`submission.reply()`するのが基本的な使い方になります。

```tsx
// action.ts
"use server";

import { parseWithZod } from "@conform-to/zod";
import { redirect } from "next/navigation";
import { User } from "./schema";

export async function saveUser(prevState: unknown, formData: FormData) {
  const submission = parseWithZod(formData, {
    schema: User,
  });

  if (submission.status !== "success") {
    return submission.reply();
  }

  redirect("/success");
}
```

formコンポーネント側では`useForm`を利用して`form`オブジェクトと`fields`オブジェクトを取得します。この際`onValidate`でvalidation挙動を設定できるので、`return parseWithZod(formData, { schema: User });`とすれば、zod schemaに従ったvalidationが行われます。

あとは適宜form要素で`form`や`fields`これらを参照することでformを組み立てるのがconformの基本的な使い方です。

```tsx
// form.tsx
"use client";

import { getFormProps, getInputProps, useForm } from "@conform-to/react";
import { parseWithZod } from "@conform-to/zod";
import { useFormState } from "react-dom";
import { saveUser } from "./action";
import { User } from "./schema";

export default function Form({ storeName }: { storeName: "session" | "url" }) {
  const [lastResult, action] = useFormState(saveUser, undefined);
  const [form, fields] = useForm({
    lastResult,
    onValidate({ formData }) {
      return parseWithZod(formData, { schema: User });
    },
  });

  return (
    <form {...getFormProps(form)} action={action} noValidate>
      <div style={{ display: "flex", columnGap: "10px" }}>
        <label htmlFor={fields.firstName.id}>First name</label>
        <input
          {...getInputProps(fields.firstName, {
            type: "text",
          })}
          key={fields.firstName.key}
        />
        <div>{fields.firstName.errors}</div>
      </div>
      <div style={{ display: "flex", columnGap: "10px", marginTop: "10px" }}>
        <label htmlFor={fields.lastName.id}>Last name</label>
        <input
          {...getInputProps(fields.lastName, {
            type: "text",
          })}
          key={fields.firstName.key}
        />
        <div>{fields.lastName.errors}</div>
      </div>
      <div style={{ display: "flex", columnGap: "10px" }}>
        <button type="submit">submit</button>
        <button type="submit" {...form.reset.getButtonProps()}>
          Reset
        </button>
      </div>
    </form>
  );
}
```

実際にこれで作った画面は以下のようになります。

_初期状態_
![pure conform 0](/images/location-state-conform/pure-conform-0.png)

_入力後_
![pure conform 1](/images/location-state-conform/pure-conform-1.png)

しかし前述の通り、入力後にリロードやブラウザバックを行うと初期状態に戻ってしまいます。

_ブラウザバック・フォワード後_
![pure conform 2](/images/location-state-conform/pure-conform-2.png)

`@location-state/conform`を導入してリロード時やブラウザバック時の復元を実現しましょう。

### @location-state/conformを追加・実装

`@location-state/core`と`@location-state/conform`を追加します。

```bash
$ pnpm add @location-state/core @location-state/conform
```

Providerを設定する必要があるので、`app/layout.tsx`にClient ComponentsでProviderを追加します。

```tsx
// app/providers.tsx
"use client";

import { LocationStateProvider } from "@location-state/core";
import type { ReactNode } from "react";

export function Providers({ children }: { children: ReactNode }) {
  return <LocationStateProvider>{children}</LocationStateProvider>;
}
```

```tsx
// app/layout.tsx
import { Providers } from "./providers";

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

これで準備ができたので、conformとlocation-stateを統合します。`@location-state/conform`は`useLocationForm`というhooksを提供しており、`formOptions`と`getLocationFormProps`を取得できます。前者はconformの`useForm`のオプション、後者は`getFormProps`をラップした物になります。

```tsx
// form.tsx
"use client";

// ...
import { useLocationForm } from "@location-state/conform";
// ...

export default function Form({ storeName }: { storeName: "session" | "url" }) {
  // ...
  const [formOptions, getLocationFormProps] = useLocationForm({
    location: {
      name: "static-form",
      storeName,
    },
  });
  const [form, fields] = useForm({
    // ...
    ...formOptions,
  });

  return (
    <form {...getLocationFormProps(form)} action={action} noValidate>
      // ...
    </form>
  );
}
```

これだけで、ブラウザバック時にもフォームの状態が復元されるようになります。実際の挙動を確認してみましょう。

_入力時_

![location-state conform 0](/images/location-state-conform/location-conform-0.png)

_ブラウザバック・フォワード後_

![location-state conform 1](/images/location-state-conform/location-conform-1.png)

ちゃんと入力してた値が復元されています。

### 動的formの対応

conformは動的にフィールドを追加するようなformにも対応しています。`@location-state/conform`も同様に対応しています。

使い方は上記のような静的なformと変わらないですが、exampleに実装があるので必要な方は参考にしてみてください。

https://github.com/recruit-tech/location-state/blob/0bad20cf44c184f6853845aca994ee685b488f9c/apps/example-next-conform/src/app/forms/%5BstoreName%5D/dynamic-form/form.tsx

## 感想

開発中、formが空になる体験はやっぱりかなり辛いなぁと改めて感じました。多くの方がブラウザバックのことをあまり気にせず実装していると思うのですが、ユーザーにとってはかなり重要な体験だと思います。

この気持ちを減らすべく、location-stateがもっと多くの人に使ってもらえたら嬉しいです。
