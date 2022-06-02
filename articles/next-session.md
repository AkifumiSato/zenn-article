---
title: "Next.jsと型安全session"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: true
---

Next.jsをBFFサーバーで使う時、セッションを使いたいケースもあるかと思います。この際に[next-session](https://github.com/hoangvvo/next-session) が結構便利で一工夫すれば型安全なセッション管理ができるので紹介です。

## next-sessionのメリット

expressでRedisなどを利用してセッション管理する例はGoogleで調べれば結構出てきます。Next.jsでもexpressをカスタムサーバーとして利用すれば、expressのエコシステムが利用できるのでNext.jsでセッション管理をしたいならこれも1つの案です。一方で`next-session`を利用する場合にはexpressを必要としないので、expressの実装や設定が当然不要だったり、依存関係を減らせるというメリットがあります。

## next-sessionの導入

installはいつものやつです。

```
// NPM
npm install next-session
// Yarn
yarn add next-session
```

`next-session`でセッションを利用するには以下の実装が必要になります。

- sessionのファクトリー関数(本稿における`getSession`)の作成
- プロダクション利用では必須オプションである`store`(`SessionStore`)の実装

前者は共通の設定などを渡しておくために必要な作業で、後者はRedisなどの外部Storeを想定しているため必要な作業です。

### getSessionの実装

`getSession`は公式通りだと以下のようになっています。

```js
// ./lib/get-session.js
import nextSession from "next-session";
export const getSession = nextSession(options);
```

この`getSession`を利用して、API Routsやpagesで以下のように利用できます。

_API Routes_
```ts
import { getSession } from "./lib/get-session.js";

export default async function handler(req, res) {
  const session = await getSession(req, res);
  session.views = session.views ? session.views + 1 : 1;
  // Also available under req.session:
  // req.session.views = req.session.views ? req.session.views + 1 : 1;
  res.send(
    `In this session, you have visited this website ${session.views} time(s).`
  );
}
```

_pages_
```tsx
import { getSession } from "./lib/get-session.js";

export default function Page({ views }) {
  return (
    <div>In this session, you have visited this website {views} time(s).</div>
  );
}

export async function getServerSideProps({ req, res }) {
  const session = await getSession(req, res);
  session.views = session.views ? session.views + 1 : 1;
  // Also available under req.session:
  // req.session.views = req.session.views ? req.session.views + 1 : 1;
  return {
    props: {
      views: session.views,
    },
  };
}
```

### getSessionに型をつける

公式のサンプル実装のままでももちろん良いのですが、このままだと実際に利用する際の`session`にどんな値がアプリケーションから設定されてるか定義されておらず、`[key: string]: any`になってしまいます。これに型をつけていきましょう。

まず`nextSession`の型定義を確認してみましょう。

```ts
// lib/session.d.ts
export default function session(options?: Options): (req: IncomingMessage & {
  session?: Session;
}, res: ServerResponse) => Promise<Session>;
// lib/type.d.ts
export declare type SessionData = {
  [key: string]: any;
  cookie: Cookie;
};
export interface Session extends SessionData {
    id: string;
    touch(): void;
    commit(): Promise<void>;
    destroy(): Promise<void>;
    [isNew]?: boolean;
    [isTouched]?: boolean;
    [isDestroyed]?: boolean;
}
```

`getSession`を実行すると`Promise<Session>`が得られるわけですが、`Session`や`SessionData`は独自のメソッドや`[key: string]: any;`なので実際にアプリケーション側でどんな値を入れてるのかわかりません。そのため、セッションObjectに型を付けたいなら少々工夫が必要です。

```typescript
// ./lib/get-session.ts
import nextSession from "next-session";

// ここにセッションの型を記述
export type AppSession = {
  accessDate?: Date;
};

// nextSession()の戻り値型を取得
type NextSessionInstance = ReturnType<typeof nextSession>;
// NextSessionInstanceの引数型を取得
type GetSessionArgs = Parameters<NextSessionInstance>;
// NextSessionInstanceの戻り値Promise<T>からTを取得し、cookieとidのみ取得
type GetSessionReturn = Pick<Awaited<ReturnType<NextSessionInstance>>, 'cookie' | 'id'>;

// getSessionの型を再定義
export const getSession: (
  ...args: GetSessionArgs
) => Promise<GetSessionReturn & AppSession> = nextSession();
```

`nextSession`は高階関数の型のみ定義されていますが、ここでは`getSession`の戻り値に型を付けたいので`nextSession`の型を分解して`getSession`の戻り値を`Promise<GetSessionReturn & AppSession>`に再定義しています。これで`AppSession`にセッションとして保持したい型を定義すれば型安全にセッションを扱えるようになりました。`getServerSideProps`などで`session`を作成してsetしようとすると、IDEなどで補完されるはずです。

また、`SessionStore`で直接利用する可能性のある`cookie`/`id`のみ抽出しているので、これらも利用可能です。**この抽出を行わないと`[k: string]: any`が残ってしまう**ので注意しましょう。

これで実際セッションを利用しようとした時に、定義してない代入や参照はエラーとなります。

```ts
export const getServerSideProps: GetServerSideProps<Props> = async ({ req, res }) => {
  const session = await getSession(req, res)
  session.name = 'Taro' // 補完が効く
  session.hoge = false // error

  return {
    props: {
      name: session.name,
    }
  }
}
```

### Session Storeの実装

次は`SessionStore`の実装になります。例としてここでは[ioredis](https://github.com/luin/ioredis) を使って実装します。`next-session`のドキュメント例が少しわかりづらいですが、`RedisStore`を`promisifyStore`に渡せば`SessionStore`型の戻り値を得られます。

```
yarn add ioredis connect-redis express-session
yarn add -D @types/connect-redis
```

```ts
// ./lib/get-session.ts
import nextSession from "next-session";
import { expressSession, promisifyStore } from "next-session/lib/compat";
import RedisStoreFactory from "connect-redis";
import Redis from "ioredis";

const RedisStore = RedisStoreFactory(expressSession);

// ここにセッションの型を記述
export type AppSession = {
  name?: string;
};

// nextSession()の戻り値型を取得
type NextSessionInstance = ReturnType<typeof nextSession>;
// NextSessionInstanceの引数型を取得
type GetSessionArgs = Parameters<NextSessionInstance>;
// NextSessionInstanceの戻り値Promise<T>からTを取得し、cookieとidのみ取得
type GetSessionReturn = Pick<Awaited<ReturnType<NextSessionInstance>>, 'cookie' | 'id'>;

// getSessionの型を再定義
export const getSession: (
  ...args: GetSessionArgs
) => Promise<GetSessionReturn & AppSession> = nextSession({
  store: promisifyStore(
    new RedisStore({
      client: new Redis(), // 必要に応じてhostやport
    })
  ),
});
```

## テスト時のセッションデータの準備

jestで`getServerSideProps`に独自の`req`などを渡してテストしたいこともあるでしょう。`next-session`は内部的には`req.session`があれば即時returnするので、これを利用するとテスト用のセッションデータの用意も簡単に行えます(若干ハックよりですが...)。

https://github.com/hoangvvo/next-session/blob/v4.0.4/src/session.ts#L56

以下`getServerSideProps`のテストの参考記事です。

https://zenn.dev/takepepe/articles/testing-gssp-and-api-routes

この記事に出てくる`gsspCtx`などで`req`を作成する際に、以下のように修正すればテスト用のセッションデータが簡単に作成できます。

```ts
export const gsspCtx = (
  ctx?: Partial<GetServerSidePropsContext>,
  session?: AppSession,
): GetServerSidePropsContext => ({
  req: createRequest({
    session: session ?? {},
  }),
  res: createResponse(),
  params: undefined,
  query: {},
  resolvedUrl: "",
  ...ctx,
});
```

## まとめ

`next-session`自体スター数もちょっと少ないし、ユーザー単位のkeyとCookieの紐付けだけ作るんだったら自前で実装しようかなと最初は思ってたんですが、実際`next-session`使うと非常に楽でした。

テストがちょっとハックよりなので、そこだけもうちょっと改善できないかが残課題ですね。
