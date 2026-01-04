````instructions
---
description: 'React コンポーネント開発のベストプラクティス'
applyTo: '**/*.{jsx,tsx,js,ts}'
---

# React コンポーネント開発ベストプラクティス

## プロジェクトコンテキスト

- React 18+ アプリケーション開発
- React 19+ で React Compiler 利用時は手動メモ化を最小化する

## コア原則

- コンポーネントは純粋関数として実装する(同じ入力 → 同じ出力)
- レンダリング中に副作用を実行しない
- Props と State を直接変更しない
- Effect は最後の手段として使用する
- Hooks はトップレベルでのみ呼び出す
- レンダリング中に作成した変数の変更は許容される

## コンポーネント設計

### 純粋性の維持

- 外部変数を変更しない
- Props のみに依存して出力を生成する
- 同じ props/state/context で常に同じ結果を返す

**❌ 悪い例**:
```jsx
let guest = 0;

function Cup() {
  guest = guest + 1;  // 副作用
  return <h2>Tea cup for guest #{guest}</h2>;
}
```

**✅ 良い例 - props のみに依存**:
```jsx
function Cup({ guest }) {
  return <h2>Tea cup for guest #{guest}</h2>;
}
```

### レンダリング中の副作用回避

- DOM 操作、API 呼び出し、タイマー設定はレンダリング中に実行しない
- 必要な値はレンダリング中に計算で取得する

**✅ 良い例**:
```jsx
function Clock({ time }) {
  const hours = time.getHours();
  const className = (hours >= 0 && hours <= 6) ? 'night' : 'day';
  return <h1 className={className}>{time.toLocaleTimeString()}</h1>;
}
```

## State 管理

### State 配置原則

- State は使用するコンポーネントの近くに配置する
- 不要な再レンダリングを防ぐ
- 常に同時更新される State はグループ化する
- Props や他の State から計算できる値は State にしない
- 同じデータを複数の State に重複させない

**✅ State 構造の良い例**:
```jsx
const [position, setPosition] = useState({ x: 0, y: 0 });
const [status, setStatus] = useState('typing'); // 'typing' | 'sending' | 'sent'
const [selectedId, setSelectedId] = useState(0); // ID のみ保持

function handlePointerMove(e) {
  setPosition({ x: e.clientX, y: e.clientY });
}
```

### State 更新パターン

```jsx
// オブジェクト更新
setPosition({ ...position, x: 100 });
setPerson({ ...person, artwork: { ...person.artwork, city: 'New Delhi' }});

// 配列更新
setItems([...items, newItem]); // 追加
setItems(items.filter(item => item.id !== id)); // 削除
setItems(items.map(item => item.id === id ? { ...item, done: !item.done } : item)); // 更新
```

### Props 変更時の State リセット

- Props 変更に応じた State リセットは `key` を使用する
- Effect は不要である

**✅ 良い例**:
```jsx
function ProfilePage({ userId }) {
  return <Profile userId={userId} key={userId} />;
}

function Profile({ userId }) {
  const [comment, setComment] = useState('');  // 自動的にリセットされる
}
```

## Effect 使用ガイドライン

### Effect が不要なケース

#### レンダリングのためのデータ変換

- レンダリング中に計算で取得する
- Effect で State を更新しない

**✅ 良い例**:
```jsx
function TodoList({ todos, filter }) {
  const visibleTodos = getFilteredTodos(todos, filter);
  return <ul>{/* ... */}</ul>;
}
```

#### ユーザーイベント処理

- イベントハンドラで直接処理する
- Effect で副作用を実行しない

**✅ 良い例**:
```jsx
function ProductPage({ product, addToCart }) {
  function buyProduct() {
    addToCart(product);
    showNotification(`Added ${product.name} to cart!`);
  }
  return <button onClick={buyProduct}>Buy</button>;
}
```

#### 親コンポーネントへの通知

- イベントハンドラで State 更新と通知を同時に実行する

**✅ 良い例**:
```jsx
function Toggle({ onChange }) {
  const [isOn, setIsOn] = useState(false);
  function updateToggle(nextIsOn) {
    setIsOn(nextIsOn);
    onChange(nextIsOn);
  }
}
```

### Effect チェーンの回避

- 複数の Effect で State 更新を連鎖させない
- イベントハンドラでバッチ更新を実行する

**❌ 悪い例**:
```jsx
// 複数の Effect で State 更新を連鎖
useEffect(() => { if (card?.gold) setGoldCardCount(c => c + 1); }, [card]);
useEffect(() => { if (goldCardCount > 3) { setRound(r => r + 1); setGoldCardCount(0); }}, [goldCardCount]);
```

**✅ 良い例**:
```jsx
function Game() {
  const [card, setCard] = useState(null);
  const [goldCardCount, setGoldCardCount] = useState(0);
  const [round, setRound] = useState(1);
  
  function handlePlaceCard(nextCard) {
    setCard(nextCard);
    if (nextCard.gold) {
      if (goldCardCount < 3) {
        setGoldCardCount(goldCardCount + 1);
      } else {
        setGoldCardCount(0);
        setRound(round + 1);
      }
    }
  }
}
```

### 外部ストアのサブスクリプション

- 外部データストアには `useSyncExternalStore` を使用する

```jsx
function useOnlineStatus() {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener('online', callback);
      window.addEventListener('offline', callback);
      return () => {
        window.removeEventListener('online', callback);
        window.removeEventListener('offline', callback);
      };
    },
    () => navigator.onLine,
    () => true
  );
}
```

## パフォーマンス最適化

### メモ化戦略

- `useMemo`: 高コストな計算をキャッシュする(1ms 以上)
- `memo`: Props が変わらない場合の再レンダリングをスキップする
- `useCallback`: メモ化された子コンポーネントに渡す関数 props をメモ化する
- React Compiler 利用時(React 19+)は手動メモ化を最小化する

```jsx
// useMemo
const visibleTodos = useMemo(() => filterTodos(todos, filter), [todos, filter]);

// memo
const TodoList = memo(function TodoList({ todos }) {
  return <ul>{todos.map(todo => <li key={todo.id}>{todo.text}</li>)}</ul>;
});

// useCallback
const handleSubmit = useCallback((orderDetails) => {
  post('/product/' + productId + '/buy', { referrer, orderDetails });
}, [productId, referrer]);
```

### 並行機能

- `useTransition`: UI の応答性を維持しながら重い処理を実行する
- `useDeferredValue`: 値の更新を遅延させる

```jsx
// useTransition
const [isPending, startTransition] = useTransition();
function handleChange(e) {
  setSearchTerm(e.target.value);  // 緊急
  startTransition(() => setResults(searchData(e.target.value)));  // 非緊急
}

// useDeferredValue
const deferredQuery = useDeferredValue(query);
const results = useMemo(() => searchData(deferredQuery), [deferredQuery]);
```

### その他の最適化

- コード分割: `React.lazy` と `Suspense` で初期バンドルサイズを削減する
- リスト仮想化: 1000+ 項目には react-window または TanStack Virtual を使用する
- 測定: 最適化前に React DevTools Profiler、Chrome DevTools、Web Vitals を使用して常に測定する

```jsx
// コード分割
const HeavyComponent = lazy(() => import('./HeavyComponent'));
<Suspense fallback={<LoadingSpinner />}><HeavyComponent /></Suspense>
```

## コード構造とパターン

### カスタム Hook の活用

- カスタム Hook を作成してロジックを再利用する

```jsx
function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(true);
  useEffect(() => {
    const updateState = () => setIsOnline(navigator.onLine);
    window.addEventListener('online', updateState);
    window.addEventListener('offline', updateState);
    return () => {
      window.removeEventListener('online', updateState);
      window.removeEventListener('offline', updateState);
    };
  }, []);
  return isOnline;
}
```

### Reducer と Context

- 複雑な State 管理には useReducer + Context を組み合わせる

```jsx
const TasksContext = createContext(null);
const TasksDispatchContext = createContext(null);

export function TasksProvider({ children }) {
  const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);
  return (
    <TasksContext.Provider value={tasks}>
      <TasksDispatchContext.Provider value={dispatch}>
        {children}
      </TasksDispatchContext.Provider>
    </TasksContext.Provider>
  );
}

export const useTasks = () => useContext(TasksContext);
export const useTasksDispatch = () => useContext(TasksDispatchContext);
```

### エクスポートパターン

- 名前付きエクスポートを優先する
- 大きなコンポーネントは 1 ファイル 1 コンポーネントにする

## React ルール

### ルール 1: コンポーネントと Hooks は純粋である必要がある

- 同じ入力に対して同じ出力(冪等性)
- 副作用はレンダリング外で実行する
- Props と State は読み取り専用である

### ルール 2: React がコンポーネントを呼び出す

コンポーネント関数を直接呼び出してはいけない。JSX として使用する。

**❌ 悪い例**:
```jsx
return <div>{MyComponent()}</div>;
```

**✅ 良い例**:
```jsx
return <div><MyComponent /></div>;
```

## React ルール

- コンポーネントと Hooks は純粋である(同じ入力 → 同じ出力、副作用はレンダリング外)
- React がコンポーネントを呼び出す: `{MyComponent()}` ではなく `<MyComponent />` を使用する
- Hooks はトップレベルでのみ: 条件分岐やループ内で呼び出さない
- Hooks は React 関数内でのみ: コンポーネントまたはカスタム Hook 内でのみ使用する

## State 管理ライブラリ

| シナリオ | 推奨 |
|----------|------|
| 小規模 | `useState` / `useReducer` |
| コンポーネント間共有 | Context API / Jotai |
| グローバル State | Zustand / Redux |
| サーバーデータ | TanStack Query |

- React の組み込み State 管理で十分な場合が多い
- 外部ライブラリは必要な場合のみ使用する

## 検証

```bash
npx tsc --noEmit  # 型チェック
npx eslint src/  # リント
npm run build    # ビルド
```

## 参考リソース

- [React 公式ドキュメント](https://react.dev)
- [React のルール](https://react.dev/reference/rules)
- [Effect が不要な場合](https://react.dev/learn/you-might-not-need-an-effect)
- [コンポーネントの純粋性を保つ](https://react.dev/learn/keeping-components-pure)

````
