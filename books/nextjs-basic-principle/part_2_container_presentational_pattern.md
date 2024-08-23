---
title: "Container/Presentationalパターン"
---

## 要約

データ取得はContainer Components・データの参照はPresentational Componentsに分離し、テスト容易性を向上させましょう。

:::message
本章の解説内容は、[Quramyさんの記事](https://quramy.medium.com/react-server-component-%E3%81%AE%E3%83%86%E3%82%B9%E3%83%88%E3%81%A8-container-presentation-separation-7da455d66576)を参考にしています。ほとんど要約した内容となりますので、より詳細に知りたい方は元記事をご参照ください。
:::

## 背景

ReactコンポーネントのテストといえばReact Testing Library(以下RTL)やStorybookなどを利用することが主流ですが、本書執筆時点でこれらのServer Components対応の状況は芳しくありません。

### React Testing Library

RTLは現状[Server Componentsに未対応](https://github.com/testing-library/react-testing-library/issues/1209)で、将来的にサポートするようなコメントも見られますが時期については不明です。

具体的には非同期なコンポーネントを`render()`することができないため、以下のようにServer Componentsのデータフェッチに依存した検証はできません。

```tsx
test("random Todo APIより取得した`dummyTodo`がタイトルとして表示される", async () => {
  // mswの設定
  server.use(
    http.get("https://dummyjson.com/todos/random", () => {
      return HttpResponse.json(dummyTodo);
    }),
  );

  await render(<TodoPage />); // `<TodoPage>`はServer Components

  expect(
    screen.getByRole("heading", { name: dummyTodo.title }),
  ).toBeInTheDocument();
});
```

### Storybook

一方Storybookはexperimentalながら[Server Components対応](https://storybook.js.org/blog/storybook-react-server-components/)を実装したとしているものの、実際にはasyncなClient Componentsをレンダリングしているにすぎず、大量のmockを必要とするため筆者はあまり実用的とは考えていません。

```tsx
export default { component: DbCard };

export const Success = {
  args: { id: 1 },
  parameters: {
    moduleMock: {
      // サーバーサイド処理の分`mock`が冗長になる
      mock: () => {
        const mock = createMock(db, "findById");
        mock.mockReturnValue(
          Promise.resolve({
            name: "Beyonce",
            img: "https://blackhistorywall.files.wordpress.com/2010/02/picture-device-independent-bitmap-119.jpg",
            tel: "+123 456 789",
            email: "b@beyonce.com",
          }),
        );
        return [mock];
      },
    },
  },
};
```

## 設計・プラクティス

前述の状況を踏まえると、テスト対象となるServer Componentsは「テストしにくいデータフェッチ部分」と「テストしやすいHTMLを表現する部分」で分離しておくことが望ましいと考えられます。

このように、データを提供する層とそれを表現する層に分離するパターンは**Container/Presentationalパターン**の再来とも言えます。

### 従来のContainer/Presentationalパターン

Container/Presentationalパターンは元々、Flux全盛だったReact初期に提唱された設計手法です。データの読み取り・振る舞い(主にFluxのaction呼び出しなど)の定義をContainer Componentsが、データを参照し表示するのはPresentational Componentsが担うという責務分割がなされていました。

少々古い記事ですが、興味のある方はDan Abramov氏の以下の記事をご参照ください。

https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0

### React Server ComponentsにおけるContainer/Presentationalパターン

React Server ComponentsにおけるContainer/Presentationalパターンは従来のものとは異なり、Container Componentsはデータフェッチなどのサーバーサイド処理のみを担います。一方Presentational Componentsは、データフェッチを含まない**Shared Components**もしくはClient Componentsを指します。

| Components     | React初期                                           | RSC時代                                                           |
| -------------- | --------------------------------------------------- | ----------------------------------------------------------------- |
| Container      | 状態参照、状態変更関数の定義                        | Server Components上でのデータフェッチなどの**サーバーサイド処理** |
| Presentational | `props`を参照してReactElementを定義する純粋関数など | **Shared Components/Client Components**                           |

Shared Componentsはサーバーモジュールに依存せず、`"use client";`もないモジュールで定義されるコンポーネントを指します。このようなコンポーネントは、Client Boundary内においてはClient Componentsとして扱われ、そうでなければServer Componentsとして扱われます。

```tsx
// `"use client";`がないモジュール
export function CompanyLinks() {
  return (
    <ul>
      <li>
        <a href="/about">About</a>
      </li>
      <li>
        <a href="/contact">Contact</a>
      </li>
    </ul>
  );
}
```

:::message
上記のような`"use client";`を含まない場合において、自身でClient Boundaryを形成するのではなく、Client Boundary内で利用することを強制したいような場合には[`client-only`](https://www.npmjs.com/package/client-only)パッケージが役立ちます。
:::

Client ComponentsやShared Componentsは従来通りRTLやStorybookで扱うことができるので、テスト容易性が向上します。一方Container Componentsはこれらのツールでレンダリング・テストすることが現状難しいので、`await ArticleContainer({ id })`のように単なる関数として実行することでテストが可能です。

### 実装例

例としてランダムなTodoを取得・表示するページをContainer/Presentationalパターンで実装してみます。

```tsx
export function TodoPagePresentation({ todo }: { todo: Todo }) {
  return (
    <>
      <h1>{todo.title}</h1>
      <pre>
        <code>{JSON.stringify(todo, null, 2)}</code>
      </pre>
    </>
  );
}
```

上記のように、Presentational Componentsはデータを受け取って表示するだけのシンプルなコンポーネントです。場合によって^[[Client Componentsのユースケース](part_2_client_components_usecase)を参照ください。]はClient Componentsにすることもあるでしょう。このようなコンポーネントのテストは従来同様RTLを使ってテストできます。

```tsx
test("`todo`として渡された値がタイトルとして表示される", () => {
  render(<TodoPagePresentation todo={dummyTodo} />);

  expect(
    screen.getByRole("heading", { name: dummyTodo.todo }),
  ).toBeInTheDocument();
});
```

一方Container Componentsについては以下のように、データの取得が主な処理となります。

```tsx
export default async function Page() {
  const res = await fetch("https://dummyjson.com/todos/random", {
    next: {
      revalidate: 0,
    },
  });
  const todo = ((res) => res.json()) as Todo;

  return <TodoPagePresentation todo={todo} />;
}
```

非同期なServer ComponentsはRTLで`render()`することができないので、単なる関数として実行して戻り値を検証します。以下は簡易的なテストケースの実装例です。

```ts
describe("todos/random APIよりデータ取得成功時", () => {
  test("TodoPresentationalにAPIより取得した値が渡される", async () => {
    // mswの設定
    server.use(
      http.get("https://dummyjson.com/todos/random", () => {
        return HttpResponse.json(dummyTodo);
      }),
    );

    const page = await Page();

    expect(page.type).toBe(TodoPagePresentation);
    expect(page.props.todo).toEqual(dummyTodo);
  });
});
```

このように、コンポーネントを通常の関数のように実行すると`type`や`props`を得ることができるので、これらを元に期待値通りかテストすることができます。

ただし、上記のように`expect(page.type).toBe(TodoPagePresentation);`とすると、ReactElementの構造に強く依存してしまうFragile(壊れやすい)なテストになってしまいます。そのため、実際には[こちらの記事](https://quramy.medium.com/react-server-component-%E3%81%AE%E3%83%86%E3%82%B9%E3%83%88%E3%81%A8-container-presentation-separation-7da455d66576#:~:text=%E3%81%8A%E3%81%BE%E3%81%912%3A%20Container%20%E3%82%B3%E3%83%B3%E3%83%9D%E3%83%BC%E3%83%8D%E3%83%B3%E3%83%88%E3%81%AE%E3%83%86%E3%82%B9%E3%83%88%E3%81%A8%20JSX%20%E3%81%AE%E6%A7%8B%E9%80%A0)にあるように、ReactElementを扱うユーティリティの作成やスナップショットテストなどを検討すると良いでしょう。

```tsx
describe("todos/random APIよりデータ取得成功時", () => {
  test("TodoPresentationalにAPIより取得した値が渡される", async () => {
    // mswの設定
    server.use(
      http.get("https://dummyjson.com/todos/random", () => {
        return HttpResponse.json(dummyTodo);
      }),
    );

    const page = await Page();

    expect(
      getProps<typeof TodoPagePresentation>(page, TodoPagePresentation),
    ).toEqual({
      todo: dummyTodo,
    });
  });
});
```

## トレードオフ

### エコシステム側が将来対応する可能性

本章では現状RTLやStorybookなどがServer Componentsに対して未成熟であることを前提にしつつ、テスト容易性を向上するための手段としてContainer/Presentationalパターンが役に立つとしています。しかし今後、RTLやStorybook側の対応状況が変わってくるとContainer/Presentationalパターンを徹底せずとも容易にテストできるようになることがあるかもしれません。

ではContainer/Presentationalパターンは将来的に不要になる可能性が高く、他にメリットがないのでしょうか？次章[_Container 1stな設計_](part_2_container_1st_design)では[_Compositionパターン_](part_2_composition_pattern)とContainer/Presentationalパターンを組み合わせた、RSCのメリットを生かしつつ手戻りの少ない設計順序を提案します。
