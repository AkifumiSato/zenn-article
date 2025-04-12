---
title: "レンダリングと宣言的UI"
---

## 要約

UIの更新処理は命令的な実装になりやすく、複雑化しがちです。Reactでは、状態の更新に応じてコンポーネントを**再レンダリング**することで、**宣言的UI**を可能にしています。

## 背景

近年フロントエンド開発に対する要求は急速に複雑化しました。フレームワークなしの場合、開発者は以下のような様々な観点を考慮しつつ、UI更新処理を実装する必要があります。

- **更新前のUI**: DOMがどういう状態にあるか
- **アプリケーションの状態**: JavaScript変数やURLのクエリパラメータなど
- **複数のイベントリスナー**: イベントリスナーの順序や依存関係
- **更新後のUI**: DOMをどういう状態にすべきか

これらを統合するUI更新処理は**命令的**になりやすく、命令的な実装はすぐに複雑化します。小さなサンプルであればうまくいくかもしれませんが、Webアプリケーションのように要求が複雑な場合、実装難易度は非常に高くなります。

### 実装例

下記はTodoリストの追加や削除を、Reactなしで実装した例です。

```html
<div>
  <form id="todoForm">
    <input type="text" id="newTodoTextInput" />
    <button type="submit">追加</button>
    <p id="todoError" role="alert" style="color: red;"></p>
  </form>
  <ul id="todoList">
    <!-- <li>...</li> -->
  </ul>
</div>
```

```js
const todoForm = document.getElementById("todoForm");
const newTodoTextInput = document.getElementById("newTodoTextInput");
const todoList = document.getElementById("todoList");
const todoError = document.getElementById("todoError");

const formSchema = z.object({
  newTodo: z.string().trim().min(1, { message: "Todoを入力してください" }),
});

todoForm.addEventListener("submit", (event) => {
  event.preventDefault();

  const validationResult = formSchema.safeParse();
  if (!validationResult.success) {
    const errorMessage =
      validationResult.error.errors[0]?.message || "入力内容が無効です";
    todoError.textContent = errorMessage;
    return;
  }

  // エラーメッセージをクリア
  todoError.textContent = "";

  // 新しいリストアイテムを作成
  const newListItem = document.createElement("li");
  newListItem.textContent = validationResult.data.newTodo;

  // 削除ボタンを作成
  const deleteButton = document.createElement("button");
  deleteButton.textContent = "削除";
  deleteButton.addEventListener("click", () => {
    todoList.removeChild(newListItem);
  });

  // リストアイテムに削除ボタンを追加
  newListItem.appendChild(deleteButton);

  // リストに新しいアイテムを追加
  todoList.appendChild(newListItem);

  // 入力フィールドをクリア
  newTodoTextInput.value = "";
});
```

上記実装では、「追加」クリック後にDOMがどういう状態になるか全体を読み通さないと想像できません。現状コード自体がそれほど長くないため問題に感じづらいかもしれませんが、ここにさらに「一括削除ボタン」やサーバー側への「保存ボタン」を追加すると、複数のイベントリスナーが`<ul id="todoList">`内のDOMを扱うことにため、DOMがいつどういう状態になっているか各処理の依存関係を把握し、影響を考慮しなければなりません。

## 設計思想

Reactでは、「フロントエンド開発における複雑さの主な要因は、UIの状態管理とそれに伴う更新処理である」という考えのもと、状態の更新に応じてコンポーネントを**再レンダリング**することでUIの更新処理を自動化し、**宣言的UI**^[参考: [宣言型 UI と命令型 UI の比較](https://ja.react.dev/learn/reacting-to-input-with-state#how-declarative-ui-compares-to-imperative)]を実現しました。宣言的UIは、React登場以降多くのフレームワークで採用されており、現代フロントエンド開発において主流となっています。

以下は前述のTodoアプリをReactで再実装した例です。

```tsx
const formSchema = z.object({
  newTodo: z.string().trim().min(1, { message: "Todoを入力してください" }),
});

type FormValues = z.infer<typeof formSchema>;

function TodoList() {
  const [todoItems, setTodoItems] = useState<string[]>([]);
  // `useForm()`はreact-hook-formのhooks
  const {
    register,
    handleSubmit,
    reset,
    formState: { errors },
  } = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: {
      newTodo: "",
    },
  });

  const onSubmit: SubmitHandler<FormValues> = (data) => {
    setTodoItems([...todoItems, data.newTodo]);
    reset();
  };

  const handleDeleteItem = (target: number) => () => {
    setTodoItems(todoItems.filter((_, index) => index !== target));
  };

  return (
    <div>
      <form onSubmit={handleSubmit(onSubmit)}>
        <input type="text" {...register("newTodo")} />
        <button type="submit">追加</button>
        {errors.newTodo && (
          <p role="alert" style={{ color: "red" }}>
            {errors.newTodo.message}
          </p>
        )}
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

命令型の実装では最終的なDOM構造が予測しづらかったのに対し、Reactでは`return`文のところを見れば明らかなので、UIの結果を予測しやすくなりました。

### 宣言的UIのメンタルモデル

**宣言的UI**とは、文字通りUIを宣言的に表現することを指しています。この文脈における「宣言的」とは、値として表現できることを指しており、宣言的UIはよく`UI = f(state)`と表現されます。この式における`f()`はコンポーネント関数、`state`は文字通り状態です。

UIを式で表現できるなら、初期描画時と`state`の更新時に式を実行すれば常にあるべきUIを得ることができます。Reactでは特に`state`の更新時にこの式を再実行することを、**再レンダリング**と呼びます^[参考: [レンダリング大全](https://zenn.dev/txxm/articles/f04b21949ddab3)]。

::::details `UI = f(state)`とReact Server Components
:::message
[React Server Components](https://ja.react.dev/reference/rsc/server-components)では上述のメンタルモデルは更新され、以下の2つが統合される形となりました。

- `UI = f(state)`: Client Components
- `UI = f(data)`: Server Components

詳細な解説はDan Abramov氏の[The Two Reacts](https://overreacted.io/the-two-reacts/)を参照ください。
:::
::::

### 再レンダリングの大まかな仕組み

Reactにおける`state`は`useState()`で定義します。`useState()`は`state`と`state`を更新するためのsetter関数を返し、この関数を通じて`state`が更新された時にReactはコンポーネントを再レンダリングします。

:::message alert
setter関数を通じずに`state`を更新しても再レンダリングはされません。必ずsetter関数を通じて`state`を更新しましょう。
:::

```tsx
export function Counter() {
  const [count, setCount] = useState(0);

  return (
    <>
      <p>count: {count}</p>
      <button onClick={() => setCount(count + 1)}>increment</button>
    </>
  );
}
```

上記の例では、クリックのたびに`setCount`が呼び出され、`<Counter>`が再レンダリングされます。

Reactは「更新前のUI」と「更新後のUI」を比較して、差分があった場合のみ実際のDOMに変更を反映します^[参考: [レンダーとコミット](https://ja.react.dev/learn/render-and-commit)]。そのため、再レンダリングの結果差分がない場合にはDOMへの適用はスキップされます。

### コンポーネントの純粋性

`UI = f(state)`に表されるように、Reactが宣言的UIを実現する上で非常に重要なのは、コンポーネントの**純粋性**です。

プログラミングにおける純粋性とは、同じ入力に対して常に同じ出力を返し、かつ関数外の状態を変更したり外部の影響を受けたりしない関数のことを指します。関数やコンポーネントが純粋でなくなる要因となる処理は**副作用**と呼ばれます。具体的には関数外の状態変更であったり、外部APIへのデータフェッチなどが副作用と見做されます。

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

Reactでは副作用を**hooks**を通じて扱うことができます。先述の`useState()`もhooksの1つで、状態の変更というUI実装における最も代表的な副作用を扱います。

他にも様々な副作用に対応するために、Reactでは[Escape Hatches](https://ja.react.dev/learn/escape-hatches)として、`useEffect()`や`useRef()`など様々なhooksが提供されています。これらは文字通りできるだけ避けるべき避難口ではありますが、現実のコンポーネント開発では必要不可欠なものでもあります。

Reactはこのように、hooksに副作用を隠蔽することでコンポーネントを純粋な関数かのように記述することができます。

## トレードオフ

### 再レンダリング範囲の適正化

Reactは状態が更新されたコンポーネント自身と、その子孫コンポーネントを再帰的にレンダリングします。そのため、上位のコンポーネントで再レンダリングを頻繁にトリガーすると、パフォーマンス的に不利になりえます。

![React Tree](/images/react-basic-principle/re-render-tree.png)

状態を上位のコンポーネントで持たざるを得ない場合には、不要なレンダリングをスキップするために[`memo`](https://ja.react.dev/reference/react/memo)の使用を検討するなどしてください。
