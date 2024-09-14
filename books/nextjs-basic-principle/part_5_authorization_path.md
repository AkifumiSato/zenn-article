---
title: "URLパスと認可"
---

## 要約

App Routerで特定のURLパス（e.g. `/dashboard`配下など）に対して認可チェックを実装するには、いくつか方法が考えられます。本書では代表的な実装パターンとして、以下3つを紹介します。

- 楽観的チェック
  - [middlewareでJWT検証](#middlewareでjwt検証)
- 厳密チェック
  - [カスタムサーバーで実装](#カスタムサーバーで実装)
  - [ページごとに処理を実装](#ページごとに処理を実装)

## 背景

「`/dashboard`配下はログイン必須」といった、特定のパス配下に対して認可チェックを儲ける（実際には認証済みチェック）ような仕様は、ごく一般的でよくある要件です。しかし、App Routerでこのような仕様を実装するには、いくつかの制約を考慮する必要があります。

### 並行レンダリングされるページとレイアウト

App Routerを学んだ方であれば、特定パス以下に共通処理を差し込むのに[レイアウト](https://nextjs.org/docs/app/building-your-application/routing/layouts-and-templates)（`layout.tsx`）が使えるかもしれないと思われた方も多いでしょう。しかし、実際にはレイアウトで認可チェックを行うことは避けるべきです。

App Routerのページとレイアウトは並行にレンダリングされるため、必ずしもレイアウト層の認可チェックがページより先に実行されるとは限りません。そのため、一見すると期待通りに動いてるようでも、RSC Payloadなどを通じて情報漏洩するリスクがあります。

詳細な解説については以下の記事が参考になります。

https://zenn.dev/moozaru/articles/0d6c4596425da9

### 制限を伴うmiddleware

上記の記事にもあるように、次に選択肢として上がるのはmiddlewareでしょう。middlewareならリクエストに対して一律処理を差し込むことができます。

ただし、Next.jsのmiddlewareはv14系現在ランタイムがedgeに限定されており、Node APIが利用できなかったりDB操作系が非推奨だったり様々な制限が伴います。

将来的にNode.jsがランタイムとして選択できるようになる可能性はありますが、現状議論中の段階です。

https://github.com/vercel/next.js/discussions/46722#discussioncomment-10262088

## 設計・プラクティス

上記背景を踏まえると、App Routerにおいて特定のパスに対する認可チェックを行う手段はいくつか考えられます。大きくは「楽観的チェック」と「厳密チェック」で2つに分けられます。

- 楽観的チェック
  - middlewareでJWT検証
- 厳密チェック
  - カスタムサーバーで実装
  - ページごとに処理を実装

### middlewareでJWT検証

前述の通り、middlewareはedgeランタイムで動作するため制約が大きいのが現状です。しかし、署名したcookieの内容に基づいてログイン状態を判定するのみなどの楽観的チェックであれば、middlewareで実装することができます。

以下は公式ドキュメントにある[参考実装](https://nextjs.org/docs/app/building-your-application/authentication#optimistic-checks-with-middleware-optional)をさらに簡略化したものです。

```tsx
import { NextRequest, NextResponse } from "next/server";
import { decrypt } from "@/app/lib/session";
import { cookies } from "next/headers";

const protectedRoutes = ["/dashboard"];

export default async function middleware(req: NextRequest) {
  const isProtectedRoute = protectedRoutes.includes(req.nextUrl.pathname);

  const cookie = cookies().get("session")?.value;
  const session = await decrypt(cookie);

  if (isProtectedRoute && !session?.userId) {
    return NextResponse.redirect(new URL("/login", req.nextUrl));
  }

  return NextResponse.next();
}

// Routes Middleware should not run on
export const config = {
  matcher: ["/((?!api|_next/static|_next/image|.*\\.png$).*)"],
};
```

`session`というcookieにJWTを格納し、middlewareで複合化し`userId`のあるなしのみで楽観的にログイン状態をチェックしています。JWTを検証しているので改ざんからは保護されています。

しかし、このような実装について公式ドキュメントでは以下のように述べられており、データアクセス層におけるセキュリティチェックを強く推奨しています。

> While Middleware can be useful for initial checks, it should not be your only line of defense in protecting your data. The majority of security checks should be performed as close as possible to your data source, see Data Access Layer for more information.
> <以下Deepl訳>
> Middlewareは初期チェックには有用ですが、データを保護するための唯一の防御手段であってはなりません。セキュリティ・チェックの大部分は、可能な限りデータ・ソースの近くで行うべきです。

データアクセス層で認可チェックを行うことは、FGAC（Fine Grained Access Control、より粒度の細かいアクセス制御）を実現したい時などに特に有効です。データアクセス層における保護については次の[_データアクセスと認可_](part_5_authorization_fetch)にて詳細に解説します。

### カスタムサーバーで実装

TBW: `hono`

### ページごとに処理を実装

## トレードオフ

### データフェッチに対する暗黙的制約

- 特に「ページごとのチェック処理」において、データフェッチが「特定の処理が通っているはずである」という前提になる
- つまり、暗黙的な制約のため実装ミスによる漏洩リスクは少々高いと考えられる
- これを回避するには...次章
