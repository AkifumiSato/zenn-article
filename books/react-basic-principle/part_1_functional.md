---
title: "関数型プログラミングの考え方"
---

## 要約

Reactは関数型プログラミングの考え方を随所に取り入れつつ、実装容易性とのバランスを大事にしています。

## 背景

Reactなしで動的なUIを実装する場合は、DOMに対してイベントリスナーを宣言し、イベントリスナー内でデータフェッチやDOM操作など様々な処理を行うことになります。このような**命令型**の実装はDOMやデータの状態遷移が追跡しにくいため、意図しない動作を引き起こしやすく、可読性や保守性を損なう傾向にあります。

下記はTodoリストの追加や削除を、Reactなしで実装した例です。

```html
<div>
  <form>
    <input type="text" id="newItemText" />
    <button type="submit" id="addItemButton">アイテムを追加</button>
  </form>
  <ul id="itemList">
    <!-- <li>...</li> -->
  </ul>
</div>
```

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

    // 削除ボタンにイベントリスナーを追加
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

上記実装では、「アイテムを追加」クリック後にDOMがどういう状態になるか全体を読み通さないと想像できません。現状コード自体がそれほど長くないので問題に感じづらいかもしれませんが、ここにさらに「一括削除ボタン」やサーバー側への「保存ボタン」を追加すると、複数のイベントリスナーが同じDOMを扱うため、DOMがいつどういう状態になっているかさらに想像しづらくなります。

## 設計思想

Reactでは、「フロントエンド開発における複雑さの主な要因は、UIの状態管理とそれに伴う更新処理である」という考えのもと、関数型プログラミングのアプローチを取り入れることで、これらの複雑性を排除した**宣言的UI**を実現しました。宣言的UIは、React登場以降多くのフレームワークで採用されており、現代フロントエンド開発において主流となっています。

以下は前述のTodoアプリをReactで再実装した例です。

```tsx
type FormValues = {
  newItemText: string;
};

function TodoList() {
  const [todoItems, setTodoItems] = useState<string[]>([]);
  // `useForm()`はreact-hook-formのhooks
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

命令型の実装では最終的なDOM構造が予測しづらかったのに対し、Reactでは最終的なDOM構造がどうあるべきか`return`文のところを見れば明らかなので、UIの結果を予測しやすくなりました。このように、最終的に「どうあるべきか」を宣言していくような実装こそ、宣言的UIと呼ばれる所以です。

### 宣言型プログラミング

宣言的UIという言葉は、関数型プログラミングを内包するより大きな概念である**宣言型プログラミング**より派生したものです。[Wikipedia](https://ja.wikipedia.org/wiki/%E5%AE%A3%E8%A8%80%E5%9E%8B%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0)では以下のように解説されています。

> 宣言型言語は、what the program must accomplish（何をなすべきか）方針で、副作用を排除した式や純粋関数の実装に努める。これは命令型言語の、how to accomplish it（どうなすべきか）方針で、副作用を前提にした操作的意味論下のアルゴリズム実装とよく対比される。

専門用語が多いですが要約すると、「どうするべきか」という命令的な実装ではなく、副作用をできるだけ排除し「どうなるべきか」を宣言するように設計する考え方が、宣言型プログラミングです。

### 純粋性と副作用

Reactが宣言的UIを実現する上で非常に重要なのは、コンポーネントの**純粋性**と**副作用**の取り扱いです。

プログラミングにおける純粋な関数とは、同じ入力に対して常に同じ出力を返し、かつ関数外の状態を変更したり外部の影響を受けたりしない関数のことを指します。関数やコンポーネントが純粋でなくなる要因となる処理は**副作用**と呼ばれます。具体的には関数外の状態変更であったり、外部APIへのデータフェッチなどが副作用と見做されます。

Reactは宣言的UIを採用しており、宣言的UIの前提としてコンポーネントは純粋である必要があります。しかし、WebのUIに求められる実装は副作用なしでは実現し得ないため、Reactでは`useState()`や`useEffect()`といった**hooks**^[各種hooksの詳細は、[公式ドキュメント](https://ja.react.dev/reference/react/hooks)を参照ください。]と呼ばれる特別な関数を提供しており、これらでのみ副作用を扱うことができます。Reactではhooksでのみ副作用を扱うことで、コンポーネントを純粋な関数かのように記述することができます。

```tsx
// NG: 呼び出すごとに結果が変わるような処理=副作用
let guest = 0;
function Cup() {
  guest = guest + 1;
  return <h2>guest #{guest}</h2>;
}

// OK: 副作用を含まない純粋な関数
function Cup({ guest }: { guest: number }) {
  return <h2>guest #{guest}</h2>;
}
```

### その他

Reactでは他にも、関数型プログラミングの考え方が様々取り入れられています。

#### immutable（不変性）

関数型プログラミングでは、データ構造を**immutable**に扱うことが推奨されています。immutableなデータ構造は処理の予測性を向上し、予測性の高いコードは可読性や保守性も高くなります。

Reactではimmutableを重んじるため、propsを直接変更してはいけません。stateに関しても更新する際には、必ず対応する更新関数を利用する必要があります。

https://ja.react.dev/reference/rules/components-and-hooks-must-be-pure#props-and-state-are-immutable

#### composable（合成性）

Reactの設計において、**composable**も非常に重要です。小さなコンポーネントを組み合わせて大きなUIを構築していく考え方は、まさにcomposableな設計です。

#### タプル

Reactでは**タプル**形式のデータ構造もよく利用されています。タプルとは複数の値をまとめた固定長のデータであり、JavaScriptでは配列で表現されます。Reactでは、`useState()`や`useReducer()`の戻り値などでタプルが利用されています。

```tsx
// 0番目の要素がstate、1番目の要素がsetter関数
const [state, setState] = useState(0);
```

## トレードオフ

### 徹底しない関数型プログラミング

Reactは関数型プログラミングに強く影響を受けていますが、関数型プログラミングのスタイルを徹底してはいません。Webアプリケーション開発において副作用は不可欠です。Reactは副作用を厳密に分離・管理するアプローチではなく、hooksでのみ副作用を扱うという規約によって、関数型プログラミングの原則を一部緩和することで、高い実装容易性を実現しています。

このようにReactは、関数型プログラミング自体を目的とせず、UI開発における生産性向上の手段として関数型プログラミングの要素を一部取り入れています。この実用性を重視したアプローチが、Reactの学習コストと保守コストのバランスを実現しており、広く受け入れられている理由の一つと考えられます。
