---
title: "データフェッチ コロケーション"
---

## 要約

データフェッチはデータを参照するコンポーネントにコロケーション^[コードをできるだけ関連性のある場所に配置することを指します。]し、コンポーネントの独立性を高めましょう。

<!-- 参考 https://kentcdodds.com/blog/colocation -->

## 背景

Pages Routerにおけるサーバーサイドでのデータフェッチは、[getServerSideProps↗︎](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props)や[getStaticProps↗︎](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-static-props)などページコンポーネントの外側で非同期関数を宣言し、Next.jsが実行結果をpropsとしてページコンポーネントに渡すという設計がなされてました。

これはいわゆる**バケツリレー**(Props Drilling)と呼ばれるpropsを親から子・孫へと渡していくような実装を必要とし、冗長で依存関係が広がりやすいというデメリットがありました。

### 実装例

以下は商品ページを想定した実装例です。APIから取得した`product`というpropsが親から孫までそのまま渡されるような実装が見受けれれます。

```tsx
type ProductProps = {
  product: Product;
};

export const getServerSideProps = (async () => {
  const res = await fetch("https://dummyjson.com/products/1");
  const product = await res.json();
  return { props: { product } };
}) satisfies GetServerSideProps<ProductProps>;

export default function ProductPage({
  product,
}: InferGetServerSidePropsType<typeof getServerSideProps>) {
  return (
    <ProductLayout>
      <ProductContents product={product} />
    </ProductLayout>
  );
}

function ProductContents({ product }: ProductProps) {
  return (
    <>
      <ProductHeader product={product} />
      <ProductDetail product={product} />
      <ProductFooter product={product} />
    </>
  );
}

// ...
```

わかりやすいよう少々大袈裟に実装していますが、こういったバケツリレー実装はPages Routerだと発生しがちな問題です。常に最上位で必要なデータを意識し末端まで流すので、コンポーネントのネストが深くなるほどバケツリレーは増えていきます。

この設計は我々開発者に常にページという単位を意識させてしまうため、コンポーネント指向な開発と親和性が低く、高い認知負荷を伴います。

## 設計・プラクティス

App RouterではServer Componentsが利用可能なので、末端のコンポーネントへ**データフェッチをコロケーション**することを推奨^[公式ドキュメントにおける[ベストプラクティス↗︎](https://nextjs.org/docs/14/app/building-your-application/data-fetching/patterns#fetching-data-where-its-needed)を参照ください。]しています。

もちろんページの実装規模にもよるので、小規模な実装であればページコンポーネントでデータフェッチしても問題はないでしょう。しかし、ページコンポーネントが肥大化していくと中間層でのバケツリレーが発生しやすくなるので、できるだけ末端のコンポーネントでデータフェッチを行うことを推奨します。

「それでは全く同じデータフェッチが何度も実行されてしまうのではないか」と懸念される方もいるかもしれませんが、App Routerでは[Request Memoization↗︎](https://nextjs.org/docs/app/guides/caching#request-memoization)によってデータフェッチがメモ化されるため、全く同じデータフェッチが複数回実行されることないように設計されています。

### 実装例

前述の商品ページの実装例をApp Routerに移行する場合、以下のような実装になるでしょう。

```tsx
type ProductProps = {
  product: Product;
};

// <ProductLayout>は`layout.tsx`へ移動
export default function ProductPage() {
  return (
    <>
      <ProductHeader />
      <ProductDetail />
      <ProductFooter />
    </>
  );
}

async function ProductHeader() {
  const res = await fetchProduct();

  return <>...</>;
}

async function ProductDetail() {
  const res = await fetchProduct();

  return <>...</>;
}

// ...

async function fetchProduct() {
  // Request Memoizationにより、実際のデータフェッチは1回しか実行されない
  const res = await fetch("https://dummyjson.com/products/1");
  return res.json();
}
```

データフェッチが各コンポーネントにコロケーションされたことで、バケツリレーがなくなりました。また、`<ProductHeader>`や`<ProductDetail>`などの子コンポーネントはそれぞれ必要な情報を自身で取得しているため、ページ全体でどんなデータフェッチを行っているか気にする必要がなくなりました。

## トレードオフ

### Request Memoizationへの理解

データフェッチのコロケーションを実現する要はRequest Memoizationなので、Request Memoizationに対する理解と最適な設計が重要になってきます。

この点については次の[Request Memoization](part_1_request_memoization)の章でより詳細に解説します。
