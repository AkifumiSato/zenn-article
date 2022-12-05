---
title: "Recoilはグローバル状態管理ライブラリなのか？"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "recoil"]
published: false
---

ReduxやRecoilを指して「グローバルな状態管理ライブラリ」とまとめてしまうことに若干違和感があったので、自分なりに考えをまとめてみました。

## Redux

[Redux](https://redux.js.org/)の歴史は状態管理ライブラリの中では古く、2015年に初版が公開されました。ReduxはReactに依存してないので、Redux単体や他のViewライブラリでも採用は可能でしたが、React×Reduxのセットは、フロントエンドに一貫した設計を設けることができるため強い人気を得ました。

Redux登場当時は、UIから状態管理を分けることが多くのエンジニアの関心ごとだったと考えられます。jQueryなどをベースに実装している場合、設計指針が統率しづらかったので、ReactとReduxを導入することで単方向データフローに設計を統一することができるというのが大きな魅力だったようにも感じます。

React×Reduxが大きな人気を得た後、多くのフロントエンジニアの関心ごとは状態を分離することから徐々に用途に応じた状態管理へと移り、昨今では[swr](https://swr.vercel.app/ja)や[react-hook-form](https://react-hook-form.com/)など用途に応じた状態管理ライブラリの人気が高まっています。

## Recoil

一方[Recoil](https://recoiljs.org/)は2020年に登場した、Metaが作成したReact専用のグローバルな状態管理ライブラリです。現在でもexperimentalながら一定の人気を得ています。Reduxと大きく異なる点として、code splittingが効くのでページ単位でのバンドルサイズを減らすことが大きく注目されました。また、`useState`に近いAPIにより、Reduxと比較するとより直感的な利用が可能です。

## グローバルな状態管理とは

ここまで、「グローバルな状態管理」という言葉にまとめてしまいましたが、ReduxとRecoilは並列に語れるものなのでしょうか？「グローバル」の解像度をもう少し上げてみましょう。

「グローバルな状態管理」には、以下2つの意味合いが混在していると筆者は考えています。

- **参照スコープ**
- **ライフタイムスコープ**

これらについては以前書いた記事でも述べています。

https://zenn.dev/akfm/articles/react-state-scope#%E3%83%A9%E3%82%A4%E3%83%95%E3%82%BF%E3%82%A4%E3%83%A0%E3%81%AB%E3%82%88%E3%82%8Bstate%E3%81%AE%E5%88%86%E9%A1%9E

Reduxは上記2つの意味においてグローバルですが、Recoilは前者に対しては必ずしもグローバルとは限りません。言い換えると、recoilの状態は必ずしも`export`せずとも利用することができます。

```tsx
const todoListState = atom({
  key: 'TodoList',
  default: [],
});

function TodoList() {
  const todoList = useRecoilValue(todoListState);

  return (
    <>
      <h1>Todo List</h1>
      {todoList.map((todoItem) => (
        <TodoItem key={todoItem.id} item={todoItem} />
      ))}
    </>
  );
}
```

上記例はページ内で複数利用があまり想定されていない例ですが、SPAであればこの場合、`useState`と異なりコンポーネントのアンマウント後も`todoListState`の値を保持することができます。

一方で`TodoList`コンポーネントを複数箇所で利用する場合、`todoListState`は共有された状態となりますが`atomFamily`などを利用すればコンポーネント単位で独立した状態管理をすることが可能です。

そして何よりこの場合、`todoListState`は`export`してないので**他ファイルで利用されることがありません**。RecoilがReduxと大きく異なるのは、利用者をこうして制限することができる点です。もちろん命名やState構造の設計次第で利用者を限定するようなことはよくあると思いますが、それを意識して運用することと仕組みでもって運用するのでは大きく意味が違います。

このことからわかるように、Recoilは必ずしも参照スコープ的にグローバルとは限りません。

| ライブラリ | 参照スコープ | ライフタイム |
| ---- | ---- | ---- |
| Redux | グローバル | グローバル |
| Recoil | **必ずしもグローバルではない** | グローバル |

## まとめ

ReduxでやれることはおおよそRecoilでも可能ですが、参照スコープをグローバルでなくすのはRecoilでなくてはできません。そのため、ライフタイムをグローバルにして扱いたいケースが多い場合には、参照スコープがグローバルで名前空間の衝突を考える必要のあるReduxよりRecoilの方が適している可能性があります。
