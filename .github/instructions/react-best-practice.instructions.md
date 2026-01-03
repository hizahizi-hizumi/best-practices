---
description: 'React コンポーネント開発のベストプラクティス規則'
applyTo: '**/*.jsx, **/*.tsx, **/*.js, **/*.ts'
---

# React コンポーネント開発ベストプラクティス

GitHub Copilot が React アプリケーション開発時に従うべきガイドラインとベストプラクティス。

## プロジェクトコンテキスト

- **対象**: React 18+ アプリケーション開発
- **スコープ**: コンポーネント設計、State 管理、パフォーマンス最適化
- **推奨環境**: React 19+（React Compiler 利用可能）

## 一般的な指示

- コンポーネントは純粋関数として実装（同じ入力に対して同じ出力）
- レンダリング中に副作用を実行しない
- Props と State を直接変更しない（イミュータブル）
- Effect は最後の手段として使用（多くの場合不要）
- Hooks のルールを必ず守る（トップレベルでのみ呼び出す）
- React Compiler（React 19+）利用時は手動メモ化を最小限に

---

## コンポーネント設計の原則

### コンポーネントの純粋性

コンポーネントは純粋関数として実装。同じ props、state、context に対して常に同じ出力を返す。

**❌ 悪い例 - 外部変数を変更**:
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

### レンダリング中の副作用を避ける

DOM 操作、API 呼び出し、タイマー設定などはレンダリング中に実行しない。

**❌ 悪い例 - レンダリング中に DOM を変更**:
```jsx
function Clock({ time }) {
  const hours = time.getHours();
  if (hours >= 0 && hours <= 6) {
    document.getElementById('time').className = 'night';  // 副作用
  }
  return <h1 id="time">{time.toLocaleTimeString()}</h1>;
}
```

**✅ 良い例 - レンダリングで計算**:
```jsx
function Clock({ time }) {
  const hours = time.getHours();
  const className = (hours >= 0 && hours <= 6) ? 'night' : 'day';
  return <h1 className={className}>{time.toLocaleTimeString()}</h1>;
}
```

### ローカルミューテーションは許容

レンダリング中に作成した変数やオブジェクトの変更は問題なし。

**✅ 良い例**:
```jsx
function TeaGathering() {
  const cups = [];
  for (let i = 1; i <= 12; i++) {
    cups.push(<Cup key={i} guest={i} />);
  }
  return cups;
}
```

---

## State 管理のベストプラクティス

### State の配置原則

State は使用するコンポーネントの近くに配置。不要な再レンダリングを防ぐ。

**❌ 悪い例 - State が高い位置にある**:
```jsx
function App() {
  const [searchTerm, setSearchTerm] = useState("");
  
  return (
    <div>
      <Header /> {/* 不要な再レンダリング */}
      <Sidebar /> {/* 不要な再レンダリング */}
      <MainContent searchTerm={searchTerm} setSearchTerm={setSearchTerm} />
    </div>
  );
}
```

**✅ 良い例 - State を使用箇所に配置**:
```jsx
function App() {
  return (
    <div>
      <Header />
      <Sidebar />
      <MainContent />
    </div>
  );
}

function MainContent() {
  const [searchTerm, setSearchTerm] = useState("");
  // State を使用
}
```

### State 構造の設計原則

#### 原則 1: 関連する State をグループ化

常に同時に更新される State は 1 つにまとめる。

**✅ 良い例**:
```jsx
const [position, setPosition] = useState({ x: 0, y: 0 });

function handlePointerMove(e) {
  setPosition({ x: e.clientX, y: e.clientY });
}
```

#### 原則 2: State の矛盾を避ける

複数の State が矛盾する可能性がある場合は、1 つにまとめる。

**❌ 悪い例 - 矛盾の可能性**:
```jsx
const [isSending, setIsSending] = useState(false);
const [isSent, setIsSent] = useState(false);
// 両方が true になる可能性
```

**✅ 良い例 - 単一の State で管理**:
```jsx
const [status, setStatus] = useState('typing'); // 'typing' | 'sending' | 'sent'
```

#### 原則 3: 冗長な State を避ける

Props や他の State から計算できる値は State に入れない。

**❌ 悪い例 - 冗長な State**:
```jsx
const [firstName, setFirstName] = useState('');
const [lastName, setLastName] = useState('');
const [fullName, setFullName] = useState('');  // 冗長
```

**✅ 良い例 - レンダリング中に計算**:
```jsx
const [firstName, setFirstName] = useState('');
const [lastName, setLastName] = useState('');
const fullName = firstName + ' ' + lastName;  // 計算で取得
```

#### 原則 4: State の重複を避ける

同じデータを複数の State に保存しない。

**❌ 悪い例 - データの重複**:
```jsx
const [items, setItems] = useState(initialItems);
const [selectedItem, setSelectedItem] = useState(items[0]);
```

**✅ 良い例 - ID のみを保持**:
```jsx
const [items, setItems] = useState(initialItems);
const [selectedId, setSelectedId] = useState(0);
const selectedItem = items.find(item => item.id === selectedId);
```

### State 更新パターン

#### オブジェクトの更新

```jsx
// 浅い更新
setPosition({ ...position, x: 100 });

// ネストしたオブジェクトの更新
setPerson({
  ...person,
  artwork: {
    ...person.artwork,
    city: 'New Delhi'
  }
});
```

#### 配列の更新

```jsx
// 追加
setItems([...items, newItem]);

// 削除
setItems(items.filter(item => item.id !== id));

// 更新
setItems(items.map(item => 
  item.id === id ? { ...item, done: !item.done } : item
));
```

### Props 変更時の State リセット

Props の変更に応じて State をリセットする場合は `key` を使用。Effect は不要。

**❌ 悪い例 - Effect を使用**:
```jsx
function ProfilePage({ userId }) {
  const [comment, setComment] = useState('');
  
  useEffect(() => {
    setComment('');
  }, [userId]);
  // ...
}
```

**✅ 良い例 - key を使用**:
```jsx
function ProfilePage({ userId }) {
  return <Profile userId={userId} key={userId} />;
}

function Profile({ userId }) {
  const [comment, setComment] = useState('');  // 自動リセット
  // ...
}
```

---

## Effect の適切な使用

### Effect が不要なケース

#### ケース 1: レンダリング用のデータ変換

**❌ 悪い例 - Effect を使用**:
```jsx
function TodoList({ todos, filter }) {
  const [visibleTodos, setVisibleTodos] = useState([]);
  
  useEffect(() => {
    setVisibleTodos(getFilteredTodos(todos, filter));
  }, [todos, filter]);
  
  return <ul>{/* ... */}</ul>;
}
```

**✅ 良い例 - レンダリング中に計算**:
```jsx
function TodoList({ todos, filter }) {
  const visibleTodos = getFilteredTodos(todos, filter);
  return <ul>{/* ... */}</ul>;
}
```

#### ケース 2: ユーザーイベントの処理

**❌ 悪い例 - Effect を使用**:
```jsx
function ProductPage({ product }) {
  useEffect(() => {
    if (product.isInCart) {
      showNotification(`Added ${product.name} to cart!`);
    }
  }, [product]);
}
```

**✅ 良い例 - イベントハンドラで処理**:
```jsx
function ProductPage({ product, addToCart }) {
  function buyProduct() {
    addToCart(product);
    showNotification(`Added ${product.name} to cart!`);
  }
  
  return <button onClick={buyProduct}>Buy</button>;
}
```

#### ケース 3: 親コンポーネントへの通知

**❌ 悪い例 - Effect を使用**:
```jsx
function Toggle({ onChange }) {
  const [isOn, setIsOn] = useState(false);
  
  useEffect(() => {
    onChange(isOn);
  }, [isOn, onChange]);
}
```

**✅ 良い例 - イベントハンドラで両方を更新**:
```jsx
function Toggle({ onChange }) {
  const [isOn, setIsOn] = useState(false);
  
  function updateToggle(nextIsOn) {
    setIsOn(nextIsOn);
    onChange(nextIsOn);
  }
  // ...
}
```

### Effect のチェーンを避ける

複数の Effect で State を連鎖的に更新しない。イベントハンドラで一度に処理。

**❌ 悪い例 - Effect のチェーン**:
```jsx
function Game() {
  const [card, setCard] = useState(null);
  const [goldCardCount, setGoldCardCount] = useState(0);
  const [round, setRound] = useState(1);
  
  useEffect(() => {
    if (card !== null && card.gold) {
      setGoldCardCount(c => c + 1);
    }
  }, [card]);
  
  useEffect(() => {
    if (goldCardCount > 3) {
      setRound(r => r + 1);
      setGoldCardCount(0);
    }
  }, [goldCardCount]);
}
```

**✅ 良い例 - イベントハンドラで一括更新**:
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

### 外部ストアへのサブスクリプション

外部データストアをサブスクライブする場合は `useSyncExternalStore` を使用。

**✅ 良い例**:
```jsx
import { useSyncExternalStore } from 'react';

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
    () => navigator.onLine,  // クライアント側
    () => true  // サーバー側
  );
}
```

---

## パフォーマンス最適化

### useMemo による計算のキャッシュ

高コストな計算（1ms 以上）をキャッシュ。React Compiler 利用時は不要な場合が多い。

**使用すべき条件**:
- 計算が明らかに遅い（1ms 以上）
- 依存関係が頻繁に変わらない

**✅ 良い例**:
```jsx
function TodoList({ todos, filter }) {
  const visibleTodos = useMemo(
    () => filterTodos(todos, filter),
    [todos, filter]
  );
  
  return <ul>{/* ... */}</ul>;
}
```

### memo によるコンポーネントの再レンダリングスキップ

Props が変わらない場合に再レンダリングをスキップ。

**✅ 良い例**:
```jsx
import { memo } from 'react';

const TodoList = memo(function TodoList({ todos }) {
  return (
    <ul>
      {todos.map(todo => <li key={todo.id}>{todo.text}</li>)}
    </ul>
  );
});
```

### useCallback による関数のメモ化

Props として渡す関数をメモ化。memo でラップされた子コンポーネントへの props に使用。

**✅ 良い例**:
```jsx
function ProductPage({ productId, referrer }) {
  const handleSubmit = useCallback((orderDetails) => {
    post('/product/' + productId + '/buy', {
      referrer,
      orderDetails
    });
  }, [productId, referrer]);
  
  return <Form onSubmit={handleSubmit} />;
}
```

### Concurrent Features

#### useTransition - 緊急度の低い更新をマーク

UI の応答性を維持しながら、重い処理を実行。

**✅ 良い例**:
```jsx
function SearchResults() {
  const [isPending, startTransition] = useTransition();
  const [searchTerm, setSearchTerm] = useState('');
  const [results, setResults] = useState([]);
  
  function handleChange(e) {
    const value = e.target.value;
    setSearchTerm(value);  // 緊急
    
    startTransition(() => {
      setResults(searchData(value));  // 非緊急
    });
  }
  
  return (
    <>
      <input value={searchTerm} onChange={handleChange} />
      {isPending ? <Spinner /> : <ResultsList results={results} />}
    </>
  );
}
```

#### useDeferredValue - 値の更新を遅延

**✅ 良い例**:
```jsx
function SearchPage({ query }) {
  const deferredQuery = useDeferredValue(query);
  const results = useMemo(() => searchData(deferredQuery), [deferredQuery]);
  
  return <ResultsList results={results} />;
}
```

### コード分割と遅延ロード

React.lazy で初期バンドルサイズを削減。

**✅ 良い例**:
```jsx
import { lazy, Suspense } from 'react';

const HeavyComponent = lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <HeavyComponent />
    </Suspense>
  );
}
```

### リストの仮想化

大きなリスト（1000+ アイテム）には仮想化を使用。

**推奨ライブラリ**: react-window、TanStack Virtual

**✅ 良い例**:
```jsx
import { FixedSizeList } from 'react-window';

function VirtualizedList({ items }) {
  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={50}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>{items[index].text}</div>
      )}
    </FixedSizeList>
  );
}
```

### パフォーマンスの測定

最適化する前に必ず測定を実施。

**使用ツール**:
- React DevTools Profiler
- Chrome DevTools Performance
- Web Vitals
- Lighthouse

**✅ 良い例**:
```jsx
import { Profiler } from 'react';

function onRenderCallback(id, phase, actualDuration) {
  console.log(`${id} の ${phase}: ${actualDuration}ms`);
}

function App() {
  return (
    <Profiler id="App" onRender={onRenderCallback}>
      <Main />
    </Profiler>
  );
}
```

---

## コード構造とパターン

### カスタム Hook の活用

ロジックを再利用可能にするためにカスタム Hook を作成。

**✅ 良い例**:
```jsx
function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(true);
  
  useEffect(() => {
    function updateState() {
      setIsOnline(navigator.onLine);
    }
    
    window.addEventListener('online', updateState);
    window.addEventListener('offline', updateState);
    return () => {
      window.removeEventListener('online', updateState);
      window.removeEventListener('offline', updateState);
    };
  }, []);
  
  return isOnline;
}

// 使用例
function ChatIndicator() {
  const isOnline = useOnlineStatus();
  return <div>{isOnline ? '🟢' : '🔴'}</div>;
}
```

### Reducer と Context の組み合わせ

複雑な State 管理には Reducer と Context を組み合わせる。

```jsx
// Context 作成
import { createContext, useContext, useReducer } from 'react';

const TasksContext = createContext(null);
const TasksDispatchContext = createContext(null);

// Provider
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

// カスタム Hook
export function useTasks() {
  return useContext(TasksContext);
}

export function useTasksDispatch() {
  return useContext(TasksDispatchContext);
}
```

### インポートとエクスポートのパターン

- 名前付きエクスポートを優先
- 大きなコンポーネントは 1 ファイル 1 コンポーネント
- 小さなコンポーネントは 1 ファイルに複数でも可

**✅ 良い例**:
```jsx
// Button.jsx
export function Button({ children, onClick }) {
  return <button onClick={onClick}>{children}</button>;
}

export function PrimaryButton({ children, onClick }) {
  return <button className="primary" onClick={onClick}>{children}</button>;
}
```

---

## React のルール

### ルール 1: コンポーネントと Hooks は純粋

- 同じ入力に対して同じ出力（べき等）
- 副作用はレンダリング外で実行
- Props と State は読み取り専用

### ルール 2: React がコンポーネントを呼び出す

コンポーネント関数を直接呼び出さない。JSX として使用。

**❌ 悪い例**:
```jsx
return <div>{MyComponent()}</div>;
```

**✅ 良い例**:
```jsx
return <div><MyComponent /></div>;
```

### ルール 3: Hooks はトップレベルのみ

条件文、ループ、ネストした関数内で Hooks を呼び出さない。

**❌ 悪い例**:
```jsx
function Form() {
  if (condition) {
    useState(0);  // NG
  }
  
  for (let i = 0; i < 10; i++) {
    useEffect(() => {});  // NG
  }
}
```

**✅ 良い例**:
```jsx
function Form() {
  const [count, setCount] = useState(0);
  useEffect(() => {});
}
```

### ルール 4: Hooks は React 関数内のみ

Hooks を呼び出せる場所:
- React コンポーネント内
- カスタム Hook 内

**❌ 悪い例**:
```jsx
function normalFunction() {
  useState(0);  // React 関数外で呼び出し NG
}
```

---

## State 管理ライブラリの選択

React 組み込みの State 管理で十分な場合が多い。外部ライブラリは必要な場合のみ。

| シナリオ | 推奨ソリューション |
|---------|-------------------|
| 小規模アプリケーション | `useState` / `useReducer` |
| コンポーネント間の共有 | Context API / Jotai |
| グローバル State 管理 | Zustand / Redux |
| サーバーデータ管理 | TanStack Query |

### 推奨ライブラリ

- **Zustand**: シンプルで軽量、グローバル State 管理
- **Jotai**: アトミックな State 管理、React に近い
- **TanStack Query**: サーバーデータの取得とキャッシュ
- **Redux**: 大規模アプリケーション、予測可能な State

---

## 検証とビルド

### 開発時の検証

```bash
# 型チェック（TypeScript）
npx tsc --noEmit

# リント
npx eslint src/

# フォーマット
npx prettier --check src/
```

### ビルド検証

```bash
# ビルド
npm run build

# プレビュー
npm run preview
```

---

## まとめ

### 必ず守る原則

- コンポーネントは純粋関数（同じ入力 → 同じ出力）
- Effect は最後の手段（多くの場合不要）
- State 構造を慎重に設計（冗長性と重複を避ける）
- Hooks のルールを守る（トップレベルのみ）
- Props と State を直接変更しない

### 推奨最適化手法

- useMemo: 高コストな計算のキャッシュ
- memo: 再レンダリングスキップ
- useCallback: 関数のメモ化
- React Compiler: 自動最適化（React 19+）

### 効果的なコード構造

- カスタム Hook でロジック再利用
- Reducer + Context で複雑な State 管理
- コンポーネント分割で責任を明確化
- フラットな State 構造で更新を容易に

---

## 参考リソース

- [React 公式ドキュメント](https://react.dev)
- [React のルール](https://react.dev/reference/rules)
- [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- [Keeping Components Pure](https://react.dev/learn/keeping-components-pure)
- [Choosing the State Structure](https://react.dev/learn/choosing-the-state-structure)
