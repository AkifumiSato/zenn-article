---
title: "関数型プログラミングのエッセンス"
---

## 要約

Reactは関数型プログラミングの考え方を随所に取り入れつつ、実装容易性とのバランスを大事にしています。

## 背景

クライアントサイドJavaScriptにおいて利用可能な[Web API](https://developer.mozilla.org/ja/docs/Web/API)の多くは、歴史的経緯よりイベント購読型の設計がされています。DOMに対しイベントリスナーを付与し、イベントリスナー内でDOM操作やイベントリスナーの操作を行うことが基本的な実装方法です。このような実装はいわゆる**手続き型**の実装になるため、DOMやイベントリスナーの状態を予想しづらく、実装は指数関数的に複雑化する傾向にありました。

下記は、フレームワークなしのTodoリストの実装例です。

```js
const newItemText = document.getElementById("newItemText");
const addItemButton = document.getElementById("addItemButton");
const itemList = document.getElementById("itemList");

addItemButton.addEventListener("click", () => {
  const text = newItemText.value.trim();
  if (text !== "") {
    // 新しいリストアイテムを作成
    const newListItem = document.createElement("li");
    newListItem.textContent = text;

    // 削除ボタンを作成
    const deleteButton = document.createElement("button");
    deleteButton.textContent = "削除";

    // 削除ボタンにイベントリスナーを追加 (イベント委譲を使わない場合)
    deleteButton.addEventListener("click", () => {
      itemList.removeChild(newListItem);
    });

    // リストアイテムに削除ボタンを追加
    newListItem.appendChild(deleteButton);

    // リストに新しいアイテムを追加
    itemList.appendChild(newListItem);

    // 入力フィールドをクリア
    newItemText.value = "";
  }
});
```

この実装にさらに「Todoの一括削除」や「Todoリストのサーバー側保存処理」を追加しようとしたらどうでしょう？複数のリスナーが同じDOMを扱うため、DOMがいつどういう状態になっているか想像しづらく、複雑に感じる人が多いのではないでしょうか。

SPAのようにページ全体でこのような処理をしようとすると、実装が非常に複雑になるであろうことは想像に難しくありません。

## 設計・プラクティス

ReactはUIの更新こそが複雑であると考え、関数型プログラミングのアプローチをいくつか取り入れることで**宣言的UI**を実現しました。宣言的UIは、React登場移行多くの主要フレームワークで採用されており、現代フロントエンド開発において主流となっています。

以下は前述のTodoアプリをReactで再実装した例です。

```tsx
type FormValues = {
  newItemText: string;
};

function TodoList() {
  const [todoItems, setTodoItems] = useState<string[]>([]);
  const { register, handleSubmit, reset } = useForm<FormValues>();

  const onSubmit: SubmitHandler<FormValues> = (data) => {
    if (data.newItemText.trim() !== "") {
      setTodoItems([...todoItems, data.newItemText]);
      reset();
    }
  };

  const handleDeleteItem = (target: number) => () => {
    setTodoItems(todoItems.filter((_, index) => index !== target));
  };

  return (
    <div>
      <form onSubmit={handleSubmit(onSubmit)}>
        <input type="text" {...register("newItemText")} />
        <button type="submit">アイテムを追加</button>
      </form>
      <ul>
        {todoItems.map((item, index) => (
          <li key={index}>
            {item}
            <button onClick={handleDeleteItem(index)}>削除</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

HTMLに関する記述も増えたので行数こそ増えて見えますが、あるべきDOM構造を宣言するような実装になったため、手続き型な実装で伴ったような複雑性は排除されています。

### 宣言型プログラミング

宣言的UIという言葉は、関数型プログラミングを内包するより大きな概念である**宣言型プログラミング**より派生したものです。[Wikipedia](https://ja.wikipedia.org/wiki/%E5%AE%A3%E8%A8%80%E5%9E%8B%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0)では以下のように解説されています。

> 宣言型言語は、what the program must accomplish（何をなすべきか）方針で、副作用を排除した式や純粋関数の実装に努める。これは命令型言語の、how to accomplish it（どうなすべきか）方針で、副作用を前提にした操作的意味論下のアルゴリズム実装とよく対比される。

専門用語が多いですが、要約すると「どうするべきか」ではなく「どうなるべきか」を宣言するように実装するような考え方が宣言型プログラミングです。

### 純粋性と副作用

Reactが宣言的UIを実現する上で非常に重要なのは、コンポーネントの**純粋性**と**副作用**の取り扱いです。

プログラミングにおける純粋とは、入力に対する出力が一定で他の影響を受けず影響を与えない、文字通り純粋な計算であることを指します。関数やコンポーネントが純粋でなくなるなら、その要因となる処理は副作用です。入力に対する戻り値を主たる作用と考えるため、それ以外の作用を副作用と呼びます。

具体的な副作用としては以下などが挙げられます。

- UIにおける状態
- 外部APIへのfetch
- ブラウザやNode.js固有のAPI依存

Reactは宣言的UIを採用しており、コンポーネントは何度も実行しうるため、コンポーネントの純粋性は非常に重要です。しかし、WebのUIは副作用なしでは実現し得ないため、Reactでは副作用を扱うAPIとして`useState()`や`useEffect()`といった**hooks**と呼ばれる特別な関数を提供しています。

:::message
各種hooksの詳細は、[公式ドキュメント](https://ja.react.dev/reference/react/hooks)を参照ください。
:::

hooksを介してのみ副作用を扱うことで、Reactコンポーネントは純粋性を保つことができます。

:::message
React Server Componentsの世界では、Reactはhooksを介さずデータフェッチを直接扱うことができますが、純粋性を保つよう設計されています。詳細は[_Next.jsの考え方 - Server Componentsの純粋性_](https://zenn.dev/akfm/books/nextjs-basic-principle/viewer/part_4_pure_server_components)を参照ください。
:::

### その他のエッセンス

Reactでは他にも、関数型プログラミングのエッセンスが様々取り入れられています。

- Immutability: オブジェクトや配列の不変性
- Composition: コンポーネントや関数の合成性
- Tuple: hooksの入出力に固定長な配列を使用

本書では割愛するので、興味のある方はそれぞれ公式ドキュメントなどを参照ください。

## トレードオフ

### 徹底しない関数型プログラミング

TBW

### Reactとリアクティブプログラミング

TBW
