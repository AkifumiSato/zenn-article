---
title: "Server Actions時代のformライブラリconform"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "conform", "nextjs"]
published: true
---

**conform**は[プログレッシブ・エンハンスメント](https://zenn.dev/cybozu_frontend/articles/think-about-pe)を意識して作られたReactのformライブラリです。

https://conform.guide/

[Remix](https://remix.run/)や[Next.js](https://nextjs.org/)などのフレームワークをサポートしています。[react-hook-form](https://react-hook-form.com/)のServer Actions対応は現状[検証段階](https://github.com/react-hook-form/react-hook-form/pull/11061)なのですが、conformはすでに**Server Actionsにも対応**しています。

本稿ではNext.js(App Router)におけるconformの使い方を中心に紹介します。

## Server Actions

もう散々他の記事や公式ドキュメントで紹介されていますが、簡単にServer Actionsについて復習します。

### 基本的な使い方

Server Actionsは**クライアントサイドから呼び出すことができる、サーバー側で実行される関数**です。

https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations

この機能の最も一般的なユースケースは、サーバー側のデータを変更する際に呼び出すことです。ReactはJSXにおけるformタグを拡張しており、`<form action={serverAction}>`のようにformの`action`propsにServer Actions関数を指定するなどして利用する方法がドキュメントではよく出てきます。このように`action`に直接指定することで、JS未ロード時も動作することが可能になります。

Server Actionsは`"use server";`ディレクティブによって識別されます。`"use server";`は関数スコープの先頭やファイルの先頭に記述します。

```tsx
// action.ts
"use server";

async function createUser(formData: FormData) {
  const fullname = formData.get("fullname");

  // データベース操作、cacheの再検証などの処理
}

// page.tsx
export default function Page() {
  return (
    <form action={createUser}>
      <input type="text" name="fullname" />
      ...
    </form>
  );
}
```

### `useFormState`

Server Actionsの実行結果に応じて状態を変更したくなることもあるでしょう。そのような場合には`useFormState`を利用することができます。

https://react.dev/reference/react-dom/hooks/useFormState

`useFormState(action, initialState, permalink?)`のように、`action`と`initialState`を引数に取ります。`action`はServer Actionsで、`initialState`は扱いたい状態の初期値です。以下はNext.jsのドキュメントにある例です。

```tsx
"use client";
 
import { useFormState } from "react-dom";
import { createUser } from "@/app/actions";
 
const initialState = {
  message: "",
};
 
export function Signup() {
  const [state, formAction] = useFormState(createUser, initialState)
 
  return (
    <form action={formAction}>
      <label htmlFor="email">Email</label>
      <input type="text" id="email" name="email" required />
      {/* ... */}
      <p aria-live="polite" className="sr-only">
        {state?.message}
      </p>
      <button>Sign up</button>
    </form>
  )
}
```

他にもServer Actionsの実行状態などを取得することができる`useFormStatus`もありますが、本稿では扱わないため割愛します。

https://react.dev/reference/react-dom/hooks/useFormStatus

## react-hook-form

App Router以前のNext.jsでformライブラリと言うと、筆者は[react-hook-form](https://react-hook-form.com/)を利用することが多かったです。react-hook-formを利用することで、[zod](https://zod.dev/)定義のバリデーションを容易に行うことができました。

```tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import * as z from "zod";

const schema = z.object({
  name: z.string(),
  age: z.number(),
});

type Schema = z.infer<typeof schema>;

const App = () => {
  const { register, handleSubmit } = useForm<Schema>({
    resolver: zodResolver(schema),
  });

  const onSubmit = (data: Schema) => {
    console.log(data)
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("name")} />
      <input {...register("age", { valueAsNumber: true })} type="number" />
      <input type="submit" />
    </form>
  )
};
```

しかし前述の通り、本稿執筆時点ではreact-hook-formはServer Actionsに対応していません。かと言ってreact-hook-formが使い慣れてしまった筆者にとって、`zod.parse`を自分で呼び出して結果をエラーに変換したりエラー時のハンドリングを自前で設計・実装するのは少々面倒に思えてしまい、Server Actions対応なformライブラリの台頭を待ち望んでいました。そこで最近知ったのが**conform**です。

## conform

以降はconformの特徴や使い方について説明します。

### conformの特徴

公式ドキュメントでは[conformの特徴](https://conform.guide/)として、以下が挙げられています。

- Progressive enhancement first APIs
- Type-safe field inference
- Fine-grained subscription
- Built-in accessibility helpers
- Automatic type coercion with Zod

これらを読む限りreact-hook-formからの移行先としては、とても良さそうに思えます。

### インストール

zodを使う場合、以下2つをインストールします。

```shell-session
$ pnpm add @conform-to/react @conform-to/zod
```

### Server Actions

公式ドキュメントに[Next.jsとconformの実装例](https://conform.guide/integration/nextjs)があります。ドキュメントだと説明がコメントアウトくらいしかないので、ここではもうちょっと詳細に使い方を説明します。

以下のようなzodスキーマを持つformを実装する場合を考えていきます。

```ts
// schema.ts
import { z } from "zod";

export const loginSchema = z.object({
  email: z.string().email(),
  password: z.string(),
  remember: z.boolean().optional(),
});
```

Server Actionでは`@conform-to/zod`の`parseWithZod`を利用して以下のように書くことができます。

```ts
// action.ts
"use server";

import { redirect } from "next/navigation";
import { parseWithZod } from "@conform-to/zod";
import { loginSchema } from "@/app/schema";

export async function login(prevState: unknown, formData: FormData) {
  const submission = parseWithZod(formData, {
    schema: loginSchema,
  });

  if (submission.status !== "success") {
    return submission.reply();
  }

  redirect("/dashboard");
}
```

注目すべきは`submission.status !== "success"`でバリデーション結果を判定し、エラー時は`return submission.reply();`としていることです。エラーの詳細やzodErrorのハンドリングなどなしに、ただ`submission.reply()`を返すだけで良いのがとても嬉しいところです。

参考実装なのでバリデーションが通ったら`redirect("/dashboard");`としていますが、この前にデータベース操作などの処理を挟むことも当然できます。

このServer Actionsは`useFormState`と`@conform-to/react`の`useForm`を利用してform側で呼び出すことがで、formの組み立て準備が完了します。

```tsx
// form.tsx
"use client";

import { useForm } from "@conform-to/react";
import { parseWithZod } from "@conform-to/zod";
import { useFormState } from "react-dom";
import { login } from "@/app/actions";
import { loginSchema } from "@/app/schema";

export function LoginForm() {
  const [lastResult, action] = useFormState(login, undefined);
  const [form, fields] = useForm({
    lastResult,
    onValidate({ formData }) {
      return parseWithZod(formData, { schema: loginSchema });
    },
    shouldValidate: "onBlur",
  });
  
  // action,form,filedsを参照しつつformを組み立てる
}
```

`useFormState`の部分は前述の通りですが、これによって得られるstateを`useForm`の`lastResult`に渡しています。`lastResult`自体は省略可能なので、Server Actionsの戻り値を状態として扱う必要がなければ`useFormState`含め省略してください。

`onValidate`でServer Actions側でも利用した`parseWithZod`を利用し上記のように1行書けば、サーバーサイドと同じバリデーションをクライアントサイドで実行することが可能になります。

これらによって得られた`action`/`form`/`fileds`を使って、form要素を組み立てればクライアントサイドバリデーション＋Server Actionsなformが完成です。

```tsx
export function LoginForm() {
  const [lastResult, action] = useFormState(login, undefined);
  const [form, fields] = useForm({
    lastResult,
    onValidate({ formData }) {
      return parseWithZod(formData, { schema: loginSchema });
    },
    shouldValidate: "onBlur",
  });

  return (
    <form id={form.id} onSubmit={form.onSubmit} action={action} noValidate>
      <div>
        <label htmlFor={fields.email.id}>>Email</label>
        <input type="email" name={fields.email.name} />
        <div>{fields.email.errors}</div>
      </div>
      <div>
        <label htmlFor={fields.password.id}>Password</label>
        <input type="password" name={fields.password.name} />
        <div>{fields.password.errors}</div>
      </div>
      <label>
        <div>
          <span>Remember me</span>
          <input type="checkbox" name={fields.remember.name} />
        </div>
      </label>
      <Button>Login</Button>
    </form>
  );
}
```

### カスタムエラー

zodでのバリデーションエラー以外にも、DB操作時にデータ不整合や権限エラーなどを扱うこともあるでしょう。そのような場合には`submission.reply({ formErrors: [error.message] })`のような形で、任意のエラーメッセージを渡すことができます。

```tsx
// action.ts
"use server";

import { redirect } from "next/navigation";
import { parseWithZod } from "@conform-to/zod";
import { loginSchema } from "@/app/schema";

export async function login(prevState: unknown, formData: FormData) {
  const submission = parseWithZod(formData, {
    schema: loginSchema,
  });

  if (submission.status !== "success") {
    return submission.reply();
  }

  // redirect("/dashboard");
  return submission.reply({
    formErrors: ["エラーメッセージ1", "エラーメッセージ2"],
  });
}
```

`formErrors`に指定されたエラー文字列の配列は、`useForm`の戻り値の`form.errors`で取得することができます。

```tsx
// form.tsx
"use client";

// ...

export function LoginForm() {
  // ...

  return (
    <form id={form.id} onSubmit={form.onSubmit} action={action} noValidate>
      {/* ... */}
      {form.errors && (
        <div>
          <h2>Error:</h2>
          <ul>
            {form.errors?.map((error) => (
              <li key={error}>{error}</li>
            ))}
          </ul>
        </div>
      )}
    </form>
  );
}
```

バリデーションだけでなく、サーバー側エラーメッセージの受け渡しも簡単に実装できました。

### a11yの改善

[getFormProps](https://conform.guide/api/react/getFormProps)や[getInputProps](https://conform.guide/api/react/getInputProps)を利用することで、冗長な記述やa11y関連の属性設定を実現することができます。

```tsx
// form.tsx
export function LoginForm() {
  // ...

  return (
    <form {...getFormProps(form)}>
      <div>
        <label htmlFor={fields.email.id}>Email</label>
        <input {...getInputProps(fields.email, { type: "email" })} />
        <div>{fields.email.errors}</div>
      </div>
      <div>
        <label htmlFor={fields.password.id}>Password</label>
        <input {...getInputProps(fields.password, { type: "password" })} />
        <div>{fields.password.errors}</div>
      </div>
      <button type="submit">Login</button>
      {/* ... */}
    </form>
  );
}
```

## 感想

これまではServer Actionsを使って実装するとやはり、バリデーションやエラーハンドリングの設計・実装が面倒そうだと感じることが多かったので、conformによって本格的にServer Actionsを使うメリットが大きくなるのではないかと感じました。実際に使ってみても、コンセプト・使い勝手ともに良さそうに思いました。

Server Actionsを使ってる方でformライブラリを探している方は、ぜひconformを検討することをお勧めします。
