---
title: "Container/Presentationalパターン"
---

## 要約

データ取得はContainer Components、データ参照はPresentational Componentsに分離し、テスト容易性を向上させましょう。

:::message
本章の解説内容は、[Quramyさんの記事↗︎](https://quramy.medium.com/react-server-component-%E3%81%AE%E3%83%86%E3%82%B9%E3%83%88%E3%81%A8-container-presentation-separation-7da455d66576)を参考にしています。ほとんど要約した内容となりますので、より詳細に知りたい方は元記事をご参照ください。
:::

## 背景

Reactコンポーネントのテストといえば[React Testing Library↗︎](https://testing-library.com/docs/react-testing-library/intro/)(RTL)や[Storybook↗︎](https://storybook.js.org/)などを利用することが主流ですが、本書執筆時点でこれらのRSC対応状況は芳しくありません。

### React Testing Library

RTLは現状[Server Componentsに未対応↗︎](https://github.com/testing-library/react-testing-library/issues/1209)で、将来的にサポートするようなコメントも見られますが時期については不明です。

具体的には非同期なコンポーネントを`render()`することができないため、以下のようにServer Componentsのデータフェッチに依存した検証はできません。

```tsx
test("random Todo APIより取得した`dummyTodo`がタイトルとして表示される", async () => {
  // mswの設定
  server.use(
    http.get("https://dummyjson.com/todos/random", () => {
      return HttpResponse.json(dummyTodo);
    }),
  );

  await render(<RandomTodo />); // `<RandomTodo>`はServer Components

  expect(
    screen.getByRole("heading", { name: dummyTodo.title }),
  ).toBeInTheDocument();
});
```

:::message
執筆時点では開発中ですが、将来的には[vitest-plugin-rsc↗︎](https://github.com/kasperpeulen/vitest-plugin-rsc)などを使うことでRSCのテストが可能になるかもしれません。
:::

### Storybook

一方、Storybookはexperimentalで[RSCに対応↗︎](https://storybook.js.org/blog/storybook-react-server-components/)していますが、内部的にはこれは非同期なClient Componentsをレンダリングしているにすぎず、大量のmockを必要とするため、実用性に疑問が残ります。

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

:::message
興味のある方は、当時Container/Presentationalパターンを提唱した[Dan Abramov氏の記事↗︎](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)をご参照ください。
:::

### RSCにおけるContainer/Presentationalパターン

RSCにおけるContainer/Presentationalパターンは従来のものとは異なり、Container Componentsはデータフェッチなどのサーバーサイド処理のみを担います。一方Presentational Componentsは、データフェッチを含まない**Shared Components**もしくはClient Componentsを指します。

| Components     | React初期                                           | RSC時代                                                           |
| -------------- | --------------------------------------------------- | ----------------------------------------------------------------- |
| Container      | 状態参照、状態変更関数の定義                        | Server Components上でのデータフェッチなどの**サーバーサイド処理** |
| Presentational | `props`を参照してReactElementを定義する純粋関数など | **Shared Components/Client Components**                           |

Shared Componentsはクライアントまたはサーバーでのみ使える機能に依存せず、`"use client"`もないモジュールで定義されるコンポーネントを指します。このようなコンポーネントは、Client Bundle^[参考: [クライアントとサーバーのバンドル境界](part_2_bundle_boundary)]においてはClient Componentsとして扱われ、Server BundleにおいてはServer Componentsとして扱われます。

```tsx
// `"use client"`がないモジュール
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
上記のようにClient Boundaryでないコンポーネントで、Client Bundleに含まれることを保証したい場合には[`client-only`↗︎](https://www.npmjs.com/package/client-only)パッケージを利用しましょう。
:::

Client ComponentsやShared Componentsは従来通りRTLやStorybookで扱うことができるので、**テスト容易性が向上**します。一方Container Componentsはこれらのツールでレンダリング・テストすることは現状難しいですが、`await ArticleContainer({ id })`のように単なる関数として実行することでテストが可能です。

### 実装例

[UIをツリーに分解する](part_2_container_1st_design)の実装例同様、ブログ記事情報の取得と表示を担うコンポーネントをContainer/Presentationalパターンで実装してみます。

#### Container Componentsの実装とテスト

Container Componentsではブログ記事情報の取得を行い、Presentational Componentsにデータを渡します。

```tsx:/posts/[postId]/_containers/post/container.tsx
export async function PostContainer({
  postId,
  children,
}: {
  postId: string;
  children: React.ReactNode;
}) {
  const post = await getPost(postId); // Request Memoization

  return <PostPresentation post={post}>{children}</PostPresentation>;
}
```

:::message

- `getPost(postId)`のようにデータフェッチ層を分離することで、[Request Memoization](part_1_request_memoization)による重複リクエスト排除を活用しやすくなります。
- 上記例ではContainerとPresentationalが1対1となっていますが、必ずしも1対1になるとは限りません。

:::

前述の通り、`<PostContainer>`のような非同期コンポーネントはRTLで`render()`することができないため、コンポーネントとしてテストすることはできません。そのため、単なる関数として実行して戻り値を検証します。

以下は簡易的なテストケースの実装例です。

```ts:/posts/[postId]/_containers/post/container.test.tsx
describe("PostAPIよりデータ取得成功時", () => {
  test("PostPresentationにAPIより取得した値が渡される", async () => {
    // mswの設定
    server.use(
      http.get("https://dummyjson.com/posts/postId", () => {
        return HttpResponse.json(post);
      }),
    );

    const { type, props } = await PostContainer({ postId: "1" });

    expect(type).toBe(PostPresentation);
    expect(props.post).toEqual(post);
  });
});
```

このように、コンポーネントを通常の関数のように実行すると`type`や`props`を得ることができるので、これらを元に期待値通りかテストすることができます。

ただし、上記のようなテストは`ReactElement`の構造に強く依存してしまいFlaky(壊れやすい)なテストになってしまいます。そのため、実際には[こちらの記事↗︎](https://quramy.medium.com/react-server-component-%E3%81%AE%E3%83%86%E3%82%B9%E3%83%88%E3%81%A8-container-presentation-separation-7da455d66576#:~:text=%E3%81%8A%E3%81%BE%E3%81%912%3A%20Container%20%E3%82%B3%E3%83%B3%E3%83%9D%E3%83%BC%E3%83%8D%E3%83%B3%E3%83%88%E3%81%AE%E3%83%86%E3%82%B9%E3%83%88%E3%81%A8%20JSX%20%E3%81%AE%E6%A7%8B%E9%80%A0)にあるように、`ReactElement`を扱うユーティリティの作成やスナップショットテストなどを検討すると良いでしょう。

```tsx
describe("PostAPIよりデータ取得成功時", () => {
  test("PostPresentationにAPIより取得した値が渡される", async () => {
    // mswの設定
    server.use(
      http.get("https://dummyjson.com/posts/postId", () => {
        return HttpResponse.json(post);
      }),
    );

    const container = await PostContainer({ postId: "1" });

    expect(
      getProps<typeof PostPresentation>(container, PostPresentation),
    ).toEqual({
      post,
    });
  });
});
```

#### Presentational Componentsの実装とテスト

一方Presentational Componentsは、データを受け取って表示するだけのシンプルなコンポーネントになります。

```tsx
export function PostPresentation({ post }: { post: Post }) {
  return (
    <>
      <h1>{post.title}</h1>
      <pre>
        <code>{JSON.stringify(post, null, 2)}</code>
      </pre>
    </>
  );
}
```

必要に応じて`"use client"`を宣言し、Client Componentsにすることもできます。

このようなコンポーネントは従来同様RTLやStorybookを使って、容易にテストできます。

```tsx
test("`post`として渡された値がタイトルとして表示される", () => {
  const post = {
    title: "test post",
  };
  render(<PostPresentation post={post} />);

  expect(
    screen.getByRole("heading", {
      name: "test post",
      level: 1,
    }),
  ).toBeInTheDocument();
});
```

### Container単位のディレクトリ構成例

Next.jsはファイルコロケーションを強く意識して設計されており、[Route Segment↗︎](https://nextjs.org/docs/app/getting-started/layouts-and-pages#creating-a-nested-route)で利用するコンポーネントや関数もできるだけコロケーションすることが推奨^[参考: [公式ドキュメント↗︎](https://nextjs.org/docs/app/getting-started/project-structure#colocation)]されます。上記手順で得られたページやレイアウトを構成するContainer Componentsも、同様にコロケーションすることが望ましいと考えられます。

以下は、筆者が推奨するディレクトリ構成の例です。[Private Folder↗︎](https://nextjs.org/docs/app/getting-started/project-structure#private-folders)を利用して、Container単位で`_containers`ディレクトリにコロケーションします。

```
/posts/[postId]
├── page.tsx
├── layout.tsx
└── _containers
    ├── post
    │  ├── index.tsx // Container Componentsをre export
    │  ├── container.tsx
    │  ├── presentational.tsx
    │  └── ... // その他のコンポーネントやUtilityなど
    └── user-profile
       ├── index.tsx // Container Componentsをre export
       ├── container.tsx
       ├── presentational.tsx
       └── ... // その他のコンポーネントやUtilityなど
```

コロケーションしたファイルは、外部から参照されることを想定した実質的にPublicなファイルと、Privateなファイルに分けることができます。上記の例では、`index.tsx`でContainer Componentsをre exportすることを想定しています。

## トレードオフ

### エコシステム側が将来対応する可能性

本章では、RSCに対するテストやStorybookの対応が未成熟であることを前提にしつつ、テスト容易性を向上するための手段としてContainer/Presentationalパターンが役に立つことを主張しました。しかし、今後エコシステムの状況が変わればより容易にテストできるようになることがあるかもしれません。その場合、Container/Presentationalパターンは変化するか不要になる可能性もあります。

### 広すぎるexport

Presentational ComponentsはContainer Componentsの実装詳細と捉えることもできるので、本来プライベート定義として扱うことが好ましいと考えられます。[Container単位のディレクトリ構成例](#container単位のディレクトリ構成例)では、Presentational Componentsは`presentational.tsx`で定義されます。

```
_containers
├── <Container Name> // e.g. `post-list`, `user-profile`
│  ├── index.tsx // Container Componentsをre export
│  ├── container.tsx
│  ├── presentational.tsx
│  └── ...
└── ...
```

上記の構成では`<Container Name>`の外から参照されるモジュールは`index.tsx`のみの想定です。ただ実際には、`presentational.tsx`で定義したコンポーネントもプロジェクトのどこからでも参照することができます。

このように、同一ディレクトリにおいてのみ利用することを想定したモジュール分割においては、[eslint-plugin-import-access↗︎](https://github.com/uhyo/eslint-plugin-import-access)やbiomeの[`noPrivateImports`↗︎](https://biomejs.dev/linter/rules/no-private-imports/)を利用すると予期せぬ外部からの`import`を制限することができます。

上記のようなディレクトリ設計に沿わない場合でも、Presentational ComponentsはContainer Componentsのみが利用しうる**実質的なプライベート定義**として扱うようにしましょう。
