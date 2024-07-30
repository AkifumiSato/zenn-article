---
title: "Container/Presentationalパターン"
---

## 要約

Container/Presentationalパターンを用いて責務を分離し、テスト容易性を向上させましょう。

:::message
本章の解説内容は、[Quramyさんの記事](https://quramy.medium.com/react-server-component-%E3%81%AE%E3%83%86%E3%82%B9%E3%83%88%E3%81%A8-container-presentation-separation-7da455d66576)を参考にしています。ほとんど要約した内容となりますので、より詳細に知りたい方は元記事をご参照ください。
:::

## 背景

ReactコンポーネントのテストといえばReact Testing Library(RTL)やStorybookなどを利用することが主流ですが、本書執筆時点でこれらのServer Components対応の状況は芳しくありません。

### React Testing Library

RTLは現状[Server Componentsに未対応](https://github.com/testing-library/react-testing-library/issues/1209)で、将来的にサポートするようなコメントも見られますが時期については不明です。

具体的には非同期なコンポーネントを`render()`することができないため、以下のようにServer Componentsのデータフェッチに依存した検証はできません。

```tsx
test("TodoPresentationalにAPIより取得した`dummyTodo`がタイトルとして表示される", () => {
  // mswの設定
  server.use(
    http.get("https://dummyjson.com/todos/random", () => {
      return HttpResponse.json(dummyTodo);
    }),
  );

  render(<TodoPage />); // `<TodoPage>`はServer Components

  expect(
    screen.getByRole("heading", { name: dummyTodo.title }),
  ).toBeInTheDocument();
});
```

### Storybook

一方Storybookはexperimentalながら[Server Components対応](https://storybook.js.org/blog/storybook-react-server-components/)を実装したとしているものの、実際にはasyncなClient Componentsをレンダリングしてるにすぎず、大量のmockを必要とするため筆者はあまり実用的とは考えていません。

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

前述の状況を踏まえるとRTLやStorybookで従来サポートしておりテストしやすいDOM部分と、データフェッチ部分でテスト対象を分離しておくことが、現状テスト観点では望ましいと考えられます。

このようにデータを提供する層とそれを表現する層に分離するパターンはFlux全盛だったReact初期に提唱されていた**Container/Presentationalパターン**そのものです。

### 従来のContainer/Presentationalパターン

Container/Presentationalパターンは元々、Flux全盛の時代に提唱された設計手法です。データの読み取り・振る舞いの定義(主にFluxのaction呼び出しなど)をContainer Componentsで定義して、データを参照し表示する純粋な表示層であるPresentational Componentsに渡すという責務分割がなされていました。

少々古い記事ですが、興味のある方はDan Abramov氏の以下の記事をご参照ください。

https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0

### React Server ComponentsにおけるContainer/Presentationalパターン

React Server ComponentsにおけるPresentational Componentsは、データフェッチを含まないShared Components(定義は[Compositionパターン](part_2_composite_pattern)を参照)もしくはClient Componentsを指します。これらはRTLやStorybookで扱うことができるので、テスト容易性が向上します。

一方でContainer Componentsはデータフェッチなどのサーバーサイド処理が主な責務となります。Container Componentsはレンダリングしてテストすることが現状難しいので、単なる関数として実行することでテストを行います。

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

上記のように、Presentationalなコンポーネントはデータを受け取って表示するだけのシンプルなコンポーネントです。場合によってはClient Componentsにすることもあるでしょう。このようなコンポーネントのテストは従来同様RTLを使ってテストできます。

```tsx
test("`todo`として渡された値がタイトルとして表示される", () => {
  render(<TodoPagePresentation todo={dummyTodo} />);

  expect(
    screen.getByRole("heading", { name: dummyTodo.todo }),
  ).toBeInTheDocument();
});
```

一方Container Componentsについては以下のように、データ取得を主な処理となります。

```tsx
export default async function Page() {
  const todo: Todo = await fetch("https://dummyjson.com/todos/random", {
    next: {
      revalidate: 0,
    },
  }).then((res) => res.json());

  return <TodoPagePresentation todo={todo} />;
}
```

非同期なServer ComponentsはRTLで`render()`することができないので、純粋なJavaScript関数として実行して、戻り値を元に検証します。以下は正常系のテストケースの実装例です。

```ts
describe("todos/random APIよりデータ取得成功時", () => {
  test("TodoPresentationalにAPIより取得した値が渡される", async () => {
    // mswの設定
    server.use(
      http.get("https://dummyjson.com/todos/random", () => {
        return HttpResponse.json(dummyTodo);
      }),
    );

    const { type, props } = await Page();

    expect(type).toBe(TodoPagePresentation);
    expect(props.todo).toEqual(dummyTodo);
  });
});
```

## トレードオフ

### エコシステム側が将来対応する可能性

Container/Presentationalパターンは現状RTLやStorybookなどがServer Componentsに対して未成熟であることを前提にしつつ、テスト容易性を向上するために役立つとしています。これはつまりRTLやStorybook側の対応が進んで前提が変わってくると、Container/Presentationalパターンは不要になる可能性があるということです。

RTLやStorybookの状況が直近大きく一変すると考えられる要素はなく、かつこのパターンを用いることのデメリットは特に多くないのではないかと筆者は考えていますが、将来エコシステムの対応が進んだ際には設計を再検討する必要があるかもしれません。
