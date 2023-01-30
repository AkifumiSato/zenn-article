---
title: "フロントエンドにおける「単体テストの考え方/使い方」"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["test", "jest"]
published: true
---

:::message
本稿における「単体テスト」とは自動テストにおける単体テストを指します。手動テストのことではないので、ご了承ください。
:::

[単体テストの考え方/使い方](https://book.mynavi.jp/ec/products/detail/id=134252)という本を読みました。筆者自身、「単体テストはプロダクションコードの付属」という意識がどこかにありました。この本を読んで、単体テストについてあまりに何もわかってなかったことに気付かされ、単体テストの設計はプロダクションコードの設計と同じくらい重要という意識に変わりました。何のために単体テストをやるのか、いいテストとは、「単体」とは、など多くの点で学びを得られ、また、多くのプラクティスとアンチパターンを知ることができました。

本稿はこの本を読んで得られた学びを、フロントエンド開発、特にコンポーネント開発に適用することを試みた際のまとめです。より詳細な解説を求む方には本を手に取ってもらう前提で、できるだけポイントを抑えられるようにまとめることを目指しました。本稿がフロントエンドのテストで悩む方の参考に、もしくはこの本に興味を持つきっかけになれば幸いです。

## 単体テストの考え方

主に「単体テストの考え方/使い方」の内容の超要約ですが、重要なポイントだけ記載します。

### 単体テストの目的

そもそも我々はなぜ、単体テストを必要としているのでしょう？単体テストをうまく利用すれば、多くの**退行**（regression）を検出することが可能で、これはソフトウェア開発におけるセーフティネットとなります。開発にセーフティネットを設けることで、ソフトウェア開発の持続可能性を高めることが可能です。

単体テストのしやすさから設計を評価し、より良い設計検討が得られるというのは一理ありますが、これは喜ばしい副産物に過ぎず、単体テストの目標はあくまで持続可能な開発であるとされています。

### 良い単体テストとは

一方で単体テストはプロダクションコード同様、コード自体が**負債**であると見なすことができます。テストコードの保守にはコストがかかり、もたらされる価値とコストを天秤にかけ、**良い単体テスト**のみにしていくことが求められます。

ここでいう良い単体テストとは、以下の4つの指標によって判断することが可能です。

- **退行に対する保護**
- **リファクタリングへの耐性**
- **迅速なフィードバック**
- **保守のしやすさ**

:::message
これらの指標についての詳細な解説は省略します。
気になる方は是非[単体テストの考え方/使い方](https://book.mynavi.jp/ec/products/detail/id=134252)を読んでください。
:::

### 「単体」の定義

単体テストの目的や評価軸は上記の通りですが、もう一歩単体テストについて深ぼってみます。単体テストにおける「単体」の定義については**古典学派**と**ロンドン学派**と呼ばれる解釈の違いが存在します。

| 名称 | 単体の捉え方 |
|-----------| -------- | 
| **ロンドン学派** | 1単位のコード（例: 1クラス）を単体と捉え、他の単体（クラス）はテストダブルに置き換える。モック主義とも呼ばれる。 |
| **古典学派（デトロイト学派）** | テストケースを単体として捉え、他のテストケースに影響しうる共有依存のみをテストダブルに置き換える。 |

筆者はロンドン学派の方が壊れにくいテストを書けるような印象を持っていましたが、良い単体テストの評価軸に沿って考えてみると古典学派の方が多くのコードを検証できるため退行を検知しやすく、かつ用意するテストダブルを減らせるので保守もしやすくなります。

フロントエンド界隈においても[msw](https://mswjs.io/)を利用してる方は自然と古典学派なテストを書くことが多いかと思います。

## 良い単体テストの書き方

前述の単体テストの考え方に基づいて、フロントエンド開発における良い単体テストの書き方について考えてみます。ここでは主にコンポーネントのテストを中心に見ていきますが、通常の関数においても応用が可能です。

:::message
以降では[storybook](https://storybook.js.org/)、[jest](https://jestjs.io/ja/)、[react testing libary](https://testing-library.com/docs/react-testing-library/intro/)、[msw](https://mswjs.io/)などを利用していきますが、これらの基本的な知識やAPIについては本稿では扱いません。
また、本校ではjestで単体テストを記載していますが、[vitest](https://vitest.dev/)でもほぼ同様の書き方が可能です。
:::

### storybookと単体テスト

フロントエンド開発においては[コンポーネント駆動](https://www.componentdriven.org/)な開発を取り入れてるチームも多いと思います。独立したコンポーネントの実行と検証は多くのメリットがあり、[storybook](https://storybook.js.org/)（もしくは類似のツール）はコンポーネント駆動開発を推進する上で欠かせないツールです。特にCSF3.0の登場以降、storybookはよりjestとの相性も良くなってきました。

コンポーネント駆動開発/storybook/CSF3.0などについては、以下の記事が参考になるかと思います。

https://zenn.dev/takepepe/articles/storybook-driven-development

コンポーネントの単体テストは上記の記事で紹介されている`composeStories`という関数を利用することで、以下のような責務分けが可能です。

- storybook: コンポーネントの代表的な利用シナリオと見た目の検証
- jest: 重要な振る舞いやa11y構造の検証

サンプルとして、`/api/todos`へリクエストして内容を表示する、以下のようなコンポーネントについて考えてみます。レンダリング時に[swr](https://swr.vercel.app/ja)で`GET:/api/todos`で`Todo[]`を取得・表示しています。

```tsx
import useSWR, { useSWRConfig } from "swr";
import axios from "axios";
import "./TodoApp.css";

const fetcher = (url: string) => axios.get(url).then((res) => res.data);

export type Todo = {
  id: number;
  text: string;
};

function TodoApp() {
  const { mutate } = useSWRConfig();
  const { data, error, isLoading } = useSWR<Todo[]>("/api/todos", fetcher);

  if (isLoading) return <div>loading...</div>;
  if (error)
    return <div role="alert">error: {JSON.stringify(error.message)}</div>;

  return (
    <div className="App">
      <h1>Todo</h1>
      <ul className="todoList">
        {data?.map((todo) => (
          <li key={todo.id}>
            <div>id: {todo.id}</div>
            <div>text: {todo.text}</div>
          </li>
        ))}
      </ul>
      <button onClick={() => mutate("/api/todos")}>revalidate</button>
    </div>
  );
}

export default TodoApp;
```

このTodoAppの代表的なUIパターン≒storyとして考えられるのは、fetchが成功した時と失敗した時の2つです。この2つを[msw](https://mswjs.io/)を利用しつつ、CSF3.0に沿って記述すると以下のようになります。

```ts
import { rest } from "msw";
import TodoApp from "./TodoApp";

export default { component: TodoApp };

// defaultで設定済みのmswハンドラが適用される
export const ApiSuccess = {};

export const ApiError = {
  parameters: {
    msw: [
      rest.get("/api/todos", (_req, res, ctx) =>
        res(
          ctx.status(500),
          ctx.json({
            message: "test error",
          })
        )
      ),
    ],
  },
};
```

以降はこのサンプルに対する単体テストについて考えていきます。サンプルの全体像は以下のリポジトリをご参照ください。

https://github.com/AkifumiSato/frontend-test-pattern-example

### AAAパターン

単体テストの世界には、**AAAパターン**と呼ばれるテストの記述方法が存在します。AAAは**Arrange**（準備）、**Act**（実行）、**Assert**（確認）の頭文字です。単体テストはこの3フェーズによって構成されるべきであり、同じフェーズが複数含まれるなら、それは複数の観点を検証しようとしていることを表していることの示唆であると考えられています。コンポーネントのテストにおいてはActはイベントと捉えることが可能です。そのため、Actが複数文で構成される場合はテスト観点が複数になっている可能性があります。

コンポーネントのテストもこのAAAパターンに基づいて記載することで、観点を自然と1つに絞られるような書き方が期待できます。AAAでは、それぞれのフェーズにおいてコメントを書くことを規約とすることを好む人も多く、本稿でもコメントでフェーズを記載することを推奨します。

以下はサンプルに対してAAAを適用しつつ単体テストを実装した例です。冒頭部分ではmswやtesting-libraryのセットアップなどを行なっています。また、サンプルではswrを利用している都合上、テストケースごとにキャッシュを独立させるために`renderWithNoCache`を定義しています。

```tsx
import { render, screen } from "@testing-library/react";
import { composeStories } from "@storybook/testing-react";
import userEvent from "@testing-library/user-event";
import * as stories from "./TodoApp.stories";
import { SWRConfig } from "swr";
import { rest } from "msw";
import { setupServer } from "msw/node";
import React from "react";

const server = setupServer();
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

const { ApiSuccess } = composeStories(stories);
const user = userEvent.setup();

const renderWithNoCache = (ui: React.ReactNode) =>
  render(<SWRConfig value={{ provider: () => new Map() }}>{ui}</SWRConfig>);

test("APIが呼び出され、取得結果が表示されること", async () => {
  // Arrange
  const apiRequestCall = jest.fn();
  server.use(
    rest.get("/api/todos", (req, res, ctx) => {
      apiRequestCall();
      return res(
        ctx.status(200),
        ctx.json([
          {
            id: 0,
            text: "test message",
          },
        ])
      );
    })
  );
  // Act
  renderWithNoCache(<ApiSuccess />);
  // Assert
  expect(await screen.findByText(/test message/)).toBeInTheDocument();
  expect(apiRequestCall).toHaveBeenCalledTimes(1);
});
```

筆者としては、テストケースをAAAのコメントで区切ることで、テストが読みやすく・書きやすくなったように感じています。

Arangeは場合によって省略できることもありますが、前述の通り、AAAで区切れない場合には複数の観点を検証してしまっている可能性があります。例えば、上記はレンダリング時の情報のみで検証していますが、recalidateボタンを押下した際のテストケースは上記とは分けて記載することが望ましいと考えられます。

```tsx
test("revalidateボタン押下で再度APIが呼び出されること", async () => {
  // Arrange
  const apiRequestCall = jest.fn();
  server.use(
    rest.get("/api/todos", (req, res, ctx) => {
      apiRequestCall();
      return res(
        ctx.status(200),
        ctx.json([
          {
            id: 0,
            text: "test message",
          },
        ])
      );
    })
  );
  const renderResult = renderWithNoCache(<ApiSuccess />);
  const revalidateButton = await renderResult.findByRole("button", {
    name: "revalidate",
  });
  // Act
  await user.click(revalidateButton);
  // Assert
  expect(apiRequestCall).toHaveBeenCalledTimes(2);
});
```

この場合、「`renderWithNoCache`と`user.click(revalidateButton)`で1つのActでは」と考える人もいるでしょうが、あくまで検証したいのは「`user.click(revalidateButton)`した時の挙動」なのでActは1つと考えられます。

### テストケースの命名

テストケースの命名は厳格な命名ルールに沿ったものや実装の詳細を表すのではなく、担当エンジニア以外が見てもわかりやすい命名がベストです。サンプルでは`TodoApp`の振る舞いを中心にグルーピングし、以下のようにすることなどが考えられます。

```tsx
describe("TodoApp", () => {
  test("APIが呼び出され、取得結果が表示されること", async () => {
    // ...
  });

  test("revalidateボタン押下で再度APIが呼び出されること", async () => {
    // ...
  });

  test("APIがエラーを返したら、エラーメッセージが表示されること", async () => {
    // ...
  });
});
```

コンポーネントに複数の振る舞いが含まれる場合、「xxxボタン押下時」「yyyフォーム入力時」のように観点ごとにグルーピングするとも考えられます。

### Arrangeの共通化

テストの共通化は検証していることを隠蔽してしまうこともあるので難しいところですが、AAAにおいてはArrangeやActは共通化しても一定のわかりやすさを保つことができます。`TodoApp`のテストでは`GET:/api/todos`のモックがよく行われるので、これを`setupTodoApi`関数として共通化することでArrangeの重複を減らすことができます。

```tsx
// ...

type RestGetCallback = Parameters<typeof rest.get>[1];
const setupTodoApi = (callback: RestGetCallback) => {
  const apiRequestCall = jest.fn(callback);
  server.use(
    rest.get("/api/todos", (req, res, ctx) => {
      return apiRequestCall(req, res, ctx);
    })
  );
  return {
    apiRequestCall,
  };
};

// ...

describe("TodoApp", () => {
  test("APIが呼び出され、取得結果が表示されること", async () => {
    // Arrange
    const { apiRequestCall } = setupTodoApi((_req, res, ctx) =>
      res(
        ctx.status(200),
        ctx.json([
          {
            id: 0,
            text: "test message",
          },
        ])
      )
    );
    // Act
    renderWithNoCache(<ApiSuccess />);
    // Assert
    expect(await screen.findByText(/test message/)).toBeInTheDocument();
    expect(apiRequestCall).toHaveBeenCalledTimes(1);
  });

  // ...
});
```

一方で、Assertは安易に共通化すると結果何を検証しているのかわからなくなることはよくあるので、基本的にAssertの共通化は避けるべきだと考えられます。どうしてもAssertの記述量を減らしたいときには、[Custom Matcher](https://jestjs.io/ja/docs/expect#expectextendmatchers)を検討してみてください。

ちなみに、`setupTodoApi`を定義したことで、前述の「APIがエラーを返したら、エラーメッセージが表示されること」というテストケースは以下のように実装することができます。

```tsx
test("APIがエラーを返したら、エラーメッセージが表示されること", async () => {
  // Arrange
  const { apiRequestCall } = setupTodoApi((_req, res, ctx) =>
    res(
      ctx.status(500),
      ctx.json({
        message: "unreach error",
      })
    )
  );
  // Act
  renderWithNoCache(<ApiSuccess />);
  // Assert
  expect(await screen.findByRole("alert")).toHaveTextContent(
    /Request failed/
  );
  expect(apiRequestCall).toHaveBeenCalledTimes(1);
});
```

## まとめ

「単体テストの考え方/使い方」にも書いてあるのですが、単体テストの入門書やテストフレームワークによる導入は世の中に溢れていますが、単体テストの実践的戦略や詳細な意味づけについては少ない印象です。ゆえに、筆者自身も設計ばかり重視してしまっていたような気がします。「単体テストの考え方/使い方」を通じて、単体テスト設計の難しさとその重要性を認識することができました。一方で、これをフロントエンドに適用するのはさらにもう一段考える必要があり、難しいテーマです。今回ある程度形式化できたような気もしますが、まだまだ改善の余地があるかもしれません。幸いにも、周りに有識者が多いので、アドバイスをもらいながらより精進したいと思います。

本稿が少しでも、単体テスト戦略や設計の参考になれば幸いです。
