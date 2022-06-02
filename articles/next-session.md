---
title: "Next.jsと型安全session"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

Next.jsをBFFサーバーで使う時、セッションを使いたいケースもあるかと思います。この際に[next-session](https://github.com/hoangvvo/next-session) が結構便利で一工夫すれば型安全なセッション管理ができるのですが、あまり日本語記事がないので紹介です。

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
- `SessionStore`の実装

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

まず`nextSession`の実装を確認してみましょう。`next-session`の型は以下のように定義されています。

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

`getSession`を実行すると`Promise<Session>`が得られるわけですが、`Session`や`SessionData`は独自のメソッドか`[key: string]: any;`なので実際にアプリケーション側でどんな値を入れてるのかわかりません。そのため、セッションObjectに型を付けたいなら少々工夫が必要です。

```typescript
// ./lib/get-session.ts
import nextSession from "next-session";

// ここにセッションの型を記述
type AppSession = {
  accessDate?: Date;
};

// nextSession()の戻り値型を取得
type NextSessionInstance = ReturnType<typeof nextSession>;
// NextSessionInstanceの引数型を取得
type GetSessionArgs = Parameters<NextSessionInstance>;
// NextSessionInstanceの戻り値Promise<T>からTを取得
type GetSessionReturn = Awaited<ReturnType<NextSessionInstance>>;

// getSessionの型を再定義
export const getSession: (
        ...args: GetSessionArgs
) => Promise<GetSessionReturn & AppSession> = nextSession();
```

`nextSession`は高階関数の型のみ定義されていますが、ここでは`getSession`の戻り値に型を付けたいので`nextSession`の型を分解して再定義しています。これで`AppSession`にセッションとして保持したい型を定義すれば型安全にセッションを扱えるようになりました。

### Session Storeの実装

次は`SessionStore`の実装になります。こちらは公式ドキュメントが少しわかりづらいですが、`RedisStore`を`promisifyStore`に渡せば`SessionStore`型の戻り値を得られます。

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
type AppSession = {
  name?: string;
};

// nextSession()の戻り値型を取得
type NextSessionInstance = ReturnType<typeof nextSession>;
// NextSessionInstanceの引数型を取得
type GetSessionArgs = Parameters<NextSessionInstance>;
// NextSessionInstanceの戻り値Promise<T>からTを取得
type GetSessionReturn = Awaited<ReturnType<NextSessionInstance>>;

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

- 実装
  - memory storeをRedisで実装してみる
- テスト
  - req.sessionを作っとけばOK
- 感想
  - 自前でやろうとしてたけど結構よかった
