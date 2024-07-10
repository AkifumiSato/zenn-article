---
title: "データフェッチ コロケーション"
---

## 要約

データフェッチはデータを必要とするコンポーネントにコロケーションしましょう。

:::message
コロケーションとは、「コードをできるだけ関連性のある場所に配置すること」を意味します。
参考: https://kentcdodds.com/blog/colocation
:::

## 背景

Pages Routerにおけるサーバーサイドでのデータフェッチは、[getServerSideProps](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props)や[getStaticProps](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-static-props)など、ページの外側で非同期関数を実行し、結果をpropsとしてページコンポーネントに渡すという設計がなされてました。これはいわゆる**バケツリレー**(Props Drilling)と呼ばれるpropsを親から子・孫へと渡していくような実装を必要とし、冗長で依存関係が広がりやすいというデメリットがありました。

### 実装例

以下に商品ページを想定した実装例を示します。`product`が親から孫までそのまま渡されるような実装が見受けれれます。

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

例なので少々簡略化していますが、こういったバケツリレー実装はPages Routerだと発生しがちな問題です。常に最上位で必要なデータを意識し、末端まで流すのでコンポーネントのネストが深くなるほど認知負荷が高くなっていく構図です。

## 設計・プラクティス

App Routerでは**データフェッチはコロケーションすべき**であり、できるだけ末端のコンポーネントでデータフェッチを行うことを推奨しています。

https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#fetching-data-where-its-needed

もちろんページの規模にもよるので小規模な実装であればページコンポーネントでデータフェッチしても問題はないでしょう。しかしページコンポーネントが肥大化していくと中間層でのバケツリレーが発生しやすくなるので、できるだけ末端のコンポーネントでデータフェッチを行うことを推奨します。

「それでは同じデータ取得を無駄に何度も行ってしまうのではないか」と懸念される方もいるかもしれませんが、App Routerでは[Request Memoization](https://nextjs.org/docs/app/building-your-application/caching#request-memoization)や[React Cache](https://nextjs.org/docs/app/building-your-application/caching#react-cache-function)によって、同じデータ取得を複数回行ってしまうことを防ぐことができます。これらを利用して、できるだけデータフェッチをコロケーションする設計を心がけましょう。

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
  // Request Memoizationにより、データ取得は1回のみ
  const res = await fetch("https://dummyjson.com/products/1");
  return res.json();
}
```

データフェッチが各コンポーネントにコロケーションされたことで、バケツリレーがなくなりました。`<ProductHeader>`と`<ProductDetail>`の利用者はそれぞれの責務のみを気にしてればよく、ページ単位で必要なデータフェッチを気にする必要がなくなりました。

## トレードオフ

データフェッチのコロケーションを実現する要は**Request Memoization**や**React Cache**です。特にRequest MemoizationについてはNext.jsが自動で行っているため、開発者の理解と設計が重要になってきます。この点については次の[Request Memoization](part_1_request_memoization)のチャプターで改めて考え方を示します。
