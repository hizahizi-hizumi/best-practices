# React ベストプラクティス

React でフロントエンドを作成する際の一般的なベストプラクティスをまとめたドキュメントです。

最終更新日: 2026年1月3日

---

## 目次

1. [コンポーネント設計の原則](#コンポーネント設計の原則)
2. [State 管理のベストプラクティス](#state-管理のベストプラクティス)
3. [Effect の適切な使用方法](#effect-の適切な使用方法)
4. [パフォーマンス最適化](#パフォーマンス最適化)
5. [コードの構造化とパターン](#コードの構造化とパターン)
6. [React のルール](#react-のルール)

---

## コンポーネント設計の原則

### 1. コンポーネントを純粋に保つ

**重要度**: ⭐⭐⭐

コンポーネントは純粋関数として実装する必要があります。つまり、同じ入力（props、state、context）に対して常に同じ出力を返すべきです。

**❌ 悪い例**:
```jsx
let guest = 0;

function Cup() {
  // 外部変数を変更している（副作用）
  guest = guest + 1;
  return <h2>Tea cup for guest #{guest}</h2>;
}
```

**✅ 良い例**:
```jsx
function Cup({ guest }) {
  // props のみに依存
  return <h2>Tea cup for guest #{guest}</h2>;
}
```

#### コンポーネントの純粋性のメリット

- **予測可能性**: 同じ入力に対して常に同じ出力が得られる
- **テストしやすさ**: 副作用がないため、テストが簡単
- **最適化可能**: React が安全に最適化できる
- **並行レンダリング**: React Compiler が自動最適化できる

### 2. レンダリング中は副作用を避ける

レンダリングフェーズでは、DOM の変更、API 呼び出し、タイマーの設定などの副作用を実行してはいけません。

**❌ 悪い例**:
```jsx
function Clock({ time }) {
  const hours = time.getHours();
  // レンダリング中に DOM を直接変更している
  if (hours >= 0 && hours <= 6) {
    document.getElementById('time').className = 'night';
  } else {
    document.getElementById('time').className = 'day';
  }
  return <h1 id="time">{time.toLocaleTimeString()}</h1>;
}
```

**✅ 良い例**:
```jsx
function Clock({ time }) {
  const hours = time.getHours();
  // className を計算してレンダリングで使用
  const className = (hours >= 0 && hours <= 6) ? 'night' : 'day';
  return <h1 className={className}>{time.toLocaleTimeString()}</h1>;
}
```

### 3. Props と State はイミュータブル

コンポーネントの props や state を直接変更してはいけません。常に新しいオブジェクトや配列を作成します。

**❌ 悪い例**:
```jsx
function handleClick() {
  // state を直接変更している
  items.push(newItem);
  setItems(items);
}
```

**✅ 良い例**:
```jsx
function handleClick() {
  // 新しい配列を作成
  setItems([...items, newItem]);
}
```

### 4. ローカルミューテーションは許容される

レンダリング中に作成した変数やオブジェクトの変更は問題ありません。

**✅ 良い例**:
```jsx
function TeaGathering() {
  const cups = [];  // レンダリング中に作成
  for (let i = 1; i <= 12; i++) {
    cups.push(<Cup key={i} guest={i} />);  // ローカル変数の変更は OK
  }
  return cups;
}
```

---

## State 管理のベストプラクティス

### 1. State をコンポーネントに近い位置に配置

**重要度**: ⭐⭐⭐

State は可能な限り、それを使用するコンポーネントの近くに配置します。これにより、不要な再レンダリングを防ぎ、パフォーマンスが向上します。

**❌ 悪い例**:
```jsx
// State がコンポーネントツリーの高い位置にある
function App() {
  const [searchTerm, setSearchTerm] = useState("");
  const [selectedItem, setSelectedItem] = useState(null);

  return (
    <div className="app">
      <Header /> {/* searchTerm が変わると不要に再レンダリング */}
      <Sidebar /> {/* searchTerm が変わると不要に再レンダリング */}
      <MainContent
        searchTerm={searchTerm}
        setSearchTerm={setSearchTerm}
      />
    </div>
  );
}
```

**✅ 良い例**:
```jsx
// State を使用するコンポーネントの近くに配置
function App() {
  return (
    <div className="app">
      <Header />
      <Sidebar />
      <MainContent /> {/* State は MainContent 内で管理 */}
    </div>
  );
}

function MainContent() {
  const [searchTerm, setSearchTerm] = useState("");
  const [selectedItem, setSelectedItem] = useState(null);
  // ... State を使用
}
```

### 2. State 構造の選択原則

**重要度**: ⭐⭐⭐

State を設計する際は、以下の原則に従います：

#### 原則 1: 関連する State をグループ化する

常に同時に更新される State は、1つの State 変数にまとめます。

**✅ 良い例**:
```jsx
// 座標は常に一緒に更新されるので、1つのオブジェクトにまとめる
const [position, setPosition] = useState({ x: 0, y: 0 });

function handlePointerMove(e) {
  setPosition({ x: e.clientX, y: e.clientY });
}
```

#### 原則 2: State の矛盾を避ける

複数の State が矛盾する可能性がある場合は、1つの State にまとめます。

**❌ 悪い例**:
```jsx
const [isSending, setIsSending] = useState(false);
const [isSent, setIsSent] = useState(false);
// isSending と isSent が同時に true になる可能性がある
```

**✅ 良い例**:
```jsx
const [status, setStatus] = useState('typing'); // 'typing' | 'sending' | 'sent'
```

#### 原則 3: 冗長な State を避ける

Props や他の State から計算できる値は、State に入れない。

**❌ 悪い例**:
```jsx
const [firstName, setFirstName] = useState('');
const [lastName, setLastName] = useState('');
const [fullName, setFullName] = useState('');  // 冗長

function handleFirstNameChange(e) {
  setFirstName(e.target.value);
  setFullName(e.target.value + ' ' + lastName);  // 同期が必要
}
```

**✅ 良い例**:
```jsx
const [firstName, setFirstName] = useState('');
const [lastName, setLastName] = useState('');
// レンダリング中に計算
const fullName = firstName + ' ' + lastName;
```

#### 原則 4: State の重複を避ける

同じデータを複数の State 変数に保存しない。

**❌ 悪い例**:
```jsx
const [items, setItems] = useState(initialItems);
const [selectedItem, setSelectedItem] = useState(items[0]);
// selectedItem は items の重複
```

**✅ 良い例**:
```jsx
const [items, setItems] = useState(initialItems);
const [selectedId, setSelectedId] = useState(0);
// 選択されたアイテムは計算で取得
const selectedItem = items.find(item => item.id === selectedId);
```

#### 原則 5: 深いネストを避ける

State は可能な限りフラットな構造にします。

**✅ 良い例**:
```jsx
// フラットな構造（正規化）
const [places, setPlaces] = useState({
  0: { id: 0, title: '(Root)', childIds: [1, 2, 3] },
  1: { id: 1, title: 'Earth', childIds: [4, 5] },
  // ...
});
```

### 3. State の更新パターン

#### オブジェクトの更新

**✅ 良い例**:
```jsx
// スプレッド構文を使用
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

**✅ 良い例**:
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

### 4. 外部 State 管理ライブラリの選択

**重要度**: ⭐⭐

多くの場合、React 組み込みの State 管理で十分です。外部ライブラリは本当に必要な場合のみ使用します。

#### State 管理の選択基準

| シナリオ                               | 推奨ソリューション             |
| -------------------------------------- | ------------------------------ |
| 小規模アプリケーション                 | `useState` または `useReducer` |
| コンポーネント間での State 共有        | Context API または Jotai       |
| サーバー統合なし、大量データ取得       | Zustand または Redux           |
| クライアントデータ制限、自動キャッシュ | TanStack Query                 |

#### 推奨ライブラリ

**Zustand**:
- シンプルで軽量
- React のやり方に沿っている
- グローバル State 管理に最適

**Jotai**:
- アトミックな State 管理
- React のやり方に沿っている
- コンポーネント間の State 共有に最適

**TanStack Query（旧 React Query）**:
- サーバーデータの取得とキャッシュに特化
- 自動的なデータ更新とキャッシュ管理
- API からのデータ取得に最適

**Redux**:
- 複雑なアプリケーション State 管理
- 予測可能な State 更新
- 大規模アプリケーションに適している

### 5. Props が変更されたときの State のリセット

**重要度**: ⭐⭐

Props の変更に応じて State をリセットする必要がある場合は、`key` を使用します。

**❌ 悪い例（Effect を使用）**:
```jsx
export default function ProfilePage({ userId }) {
  const [comment, setComment] = useState('');

  useEffect(() => {
    setComment('');
  }, [userId]);
  // ...
}
```

**✅ 良い例（key を使用）**:
```jsx
export default function ProfilePage({ userId }) {
  return <Profile userId={userId} key={userId} />;
}

function Profile({ userId }) {
  // userId が変わると自動的にリセットされる
  const [comment, setComment] = useState('');
  // ...
}
```

---

## Effect の適切な使用方法

### 1. Effect が不要な場合

**重要度**: ⭐⭐⭐

多くの場合、Effect は必要ありません。以下のケースでは Effect を使用しないでください。

#### ケース 1: レンダリング用のデータ変換

**❌ 悪い例**:
```jsx
function TodoList({ todos, filter }) {
  const [visibleTodos, setVisibleTodos] = useState([]);

  useEffect(() => {
    setVisibleTodos(getFilteredTodos(todos, filter));
  }, [todos, filter]);

  return <ul>{/* ... */}</ul>;
}
```

**✅ 良い例**:
```jsx
function TodoList({ todos, filter }) {
  // レンダリング中に計算
  const visibleTodos = getFilteredTodos(todos, filter);
  return <ul>{/* ... */}</ul>;
}
```

#### ケース 2: ユーザーイベントの処理

**❌ 悪い例**:
```jsx
function ProductPage({ product }) {
  useEffect(() => {
    if (product.isInCart) {
      showNotification(`Added ${product.name} to cart!`);
    }
  }, [product]);
  // ...
}
```

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

#### ケース 3: 親コンポーネントへの通知

**❌ 悪い例**:
```jsx
function Toggle({ onChange }) {
  const [isOn, setIsOn] = useState(false);

  useEffect(() => {
    onChange(isOn);
  }, [isOn, onChange]);
  // ...
}
```

**✅ 良い例**:
```jsx
function Toggle({ onChange }) {
  const [isOn, setIsOn] = useState(false);

  function updateToggle(nextIsOn) {
    setIsOn(nextIsOn);
    onChange(nextIsOn);  // イベントハンドラで両方を更新
  }
  // ...
}
```

### 2. Effect の計算チェーンを避ける

**❌ 悪い例**:
```jsx
function Game() {
  const [card, setCard] = useState(null);
  const [goldCardCount, setGoldCardCount] = useState(0);
  const [round, setRound] = useState(1);
  const [isGameOver, setIsGameOver] = useState(false);

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

  useEffect(() => {
    if (round > 5) {
      setIsGameOver(true);
    }
  }, [round]);
  // ...
}
```

**✅ 良い例**:
```jsx
function Game() {
  const [card, setCard] = useState(null);
  const [goldCardCount, setGoldCardCount] = useState(0);
  const [round, setRound] = useState(1);

  // レンダリング中に計算
  const isGameOver = round > 5;

  function handlePlaceCard(nextCard) {
    if (isGameOver) {
      throw Error('Game already ended.');
    }

    // イベントハンドラで全ての State を更新
    setCard(nextCard);
    if (nextCard.gold) {
      if (goldCardCount < 3) {
        setGoldCardCount(goldCardCount + 1);
      } else {
        setGoldCardCount(0);
        setRound(round + 1);
        if (round === 5) {
          alert('Good game!');
        }
      }
    }
  }
  // ...
}
```

### 3. 外部ストアへのサブスクリプション

外部データストアをサブスクライブする場合は、`useSyncExternalStore` を使用します。

**✅ 良い例**:
```jsx
import { useSyncExternalStore } from 'react';

function subscribe(callback) {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function useOnlineStatus() {
  return useSyncExternalStore(
    subscribe,
    () => navigator.onLine,  // クライアント側の値
    () => true  // サーバー側の値
  );
}
```

---

## パフォーマンス最適化

### 1. useMemo による計算のキャッシュ

**重要度**: ⭐⭐⭐

高コストな計算をキャッシュする場合に使用します。

**使用するべき場合**:
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

#### useMemo を使用すべきでない場合

- 計算が速い場合（< 1ms）
- すべての場所で使用する必要はない

### 2. memo によるコンポーネントの再レンダリングスキップ

**重要度**: ⭐⭐⭐

Props が変わらない場合に再レンダリングをスキップします。

**✅ 良い例**:
```jsx
import { memo } from 'react';

const TodoList = memo(function TodoList({ todos }) {
  // todos が変わらない限り再レンダリングされない
  return (
    <ul>
      {todos.map(todo => <li key={todo.id}>{todo.text}</li>)}
    </ul>
  );
});
```

#### memo と useMemo の組み合わせ

```jsx
function TodoApp({ todos, filter, theme }) {
  // visibleTodos をメモ化
  const visibleTodos = useMemo(
    () => filterTodos(todos, filter),
    [todos, filter]
  );

  // List コンポーネントは memo でラップされている
  return (
    <div className={theme}>
      <List items={visibleTodos} />
    </div>
  );
}

const List = memo(function List({ items }) {
  // items が変わらない限り再レンダリングされない
  return <ul>{/* ... */}</ul>;
});
```

### 3. useCallback による関数のメモ化

Props として渡す関数をメモ化する場合に使用します。

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

### 4. React Compiler

**重要度**: ⭐⭐⭐⭐

React 19 以降では、React Compiler が自動的にメモ化を行います。

- `useMemo`、`memo`、`useCallback` の多くが不要になる
- 手動での最適化の負担が減る
- コードがシンプルになる

### 5. Concurrent Features（並行機能）

**重要度**: ⭐⭐⭐

React 18 以降で利用可能な並行機能を使用して、ユーザー体験を向上させます。

#### useTransition

緊急度の低い State 更新をマークし、UI の応答性を維持します。

**✅ 良い例**:
```jsx
function SearchResults() {
  const [isPending, startTransition] = useTransition();
  const [searchTerm, setSearchTerm] = useState('');
  const [results, setResults] = useState([]);

  function handleChange(e) {
    const value = e.target.value;
    setSearchTerm(value);  // 緊急度の高い更新

    startTransition(() => {
      // 緊急度の低い更新
      setResults(searchData(value));
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

#### useDeferredValue

値の更新を遅延させ、UI の応答性を維持します。

**✅ 良い例**:
```jsx
function SearchPage({ query }) {
  const deferredQuery = useDeferredValue(query);

  // deferredQuery は query よりも遅れて更新される
  const results = useMemo(() =>
    searchData(deferredQuery),
    [deferredQuery]
  );

  return <ResultsList results={results} />;
}
```

### 6. コード分割と遅延ロード

**重要度**: ⭐⭐⭐

React.lazy を使用してコンポーネントを遅延ロードし、初期バンドルサイズを削減します。

**✅ 良い例**:
```jsx
import { lazy, Suspense } from 'react';

// 遅延ロード
const HeavyComponent = lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <HeavyComponent />
    </Suspense>
  );
}
```

### 7. リストの仮想化

大きなリストをレンダリングする場合は、仮想化ライブラリを使用します。

**推奨ライブラリ**:
- react-window
- react-virtualized
- TanStack Virtual

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
        <div style={style}>
          {items[index].text}
        </div>
      )}
    </FixedSizeList>
  );
}
```

### 8. JSX での無名関数を避ける

**重要度**: ⭐⭐

Props として渡す関数は、可能な限りメモ化するか、コンポーネント外で定義します。

**❌ 悪い例**:
```jsx
function List({ items }) {
  return (
    <>
      {items.map(item => (
        <Item
          key={item.id}
          onClick={() => handleClick(item.id)}  // 毎回新しい関数が作成される
        />
      ))}
    </>
  );
}
```

**✅ 良い例**:
```jsx
function List({ items }) {
  const handleClick = useCallback((id) => {
    // ハンドラ処理
  }, []);

  return (
    <>
      {items.map(item => (
        <Item
          key={item.id}
          id={item.id}
          onClick={handleClick}
        />
      ))}
    </>
  );
}
```

### 9. パフォーマンスの測定

**重要度**: ⭐⭐⭐

最適化する前に、必ず測定を行います。

**使用するツール**:
- React DevTools Profiler
- Chrome DevTools Performance タブ
- Web Vitals
- Lighthouse

**✅ 良い例**:
```jsx
import { Profiler } from 'react';

function onRenderCallback(
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime
) {
  console.log(`${id} の ${phase} フェーズにかかった時間: ${actualDuration}ms`);
}

function App() {
  return (
    <Profiler id="App" onRender={onRenderCallback}>
      <Navigation />
      <Main />
    </Profiler>
  );
}
```

---

## コードの構造化とパターン

### 1. カスタム Hook の活用

**重要度**: ⭐⭐

ロジックを再利用可能にするために、カスタム Hook を作成します。

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

### 2. Reducer と Context の組み合わせ

**重要度**: ⭐⭐

複雑な State 管理には、Reducer と Context を組み合わせます。

**ステップ 1: Context を作成**

```jsx
// TasksContext.js
import { createContext } from 'react';

export const TasksContext = createContext(null);
export const TasksDispatchContext = createContext(null);
```

**ステップ 2: Provider を作成**

```jsx
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
```

**ステップ 3: カスタム Hook を作成**

```jsx
export function useTasks() {
  return useContext(TasksContext);
}

export function useTasksDispatch() {
  return useContext(TasksDispatchContext);
}
```

**ステップ 4: コンポーネントで使用**

```jsx
function TaskList() {
  const tasks = useTasks();
  const dispatch = useTasksDispatch();

  return (
    <ul>
      {tasks.map(task => (
        <li key={task.id}>
          {task.text}
          <button onClick={() => {
            dispatch({ type: 'deleted', id: task.id });
          }}>
            Delete
          </button>
        </li>
      ))}
    </ul>
  );
}
```

### 3. コンポーネントのインポートとエクスポート

**ベストプラクティス**:
- デフォルトエクスポートよりも名前付きエクスポートを優先
- 1ファイルに複数の小さなコンポーネントがある場合は、すべて名前付きエクスポート
- 大きなコンポーネントは1ファイル1コンポーネント

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

### 1. コンポーネントと Hooks は純粋でなければならない

**重要度**: ⭐⭐⭐

- コンポーネントはべき等（同じ入力に対して同じ出力）
- 副作用はレンダリング外で実行
- Props と State はイミュータブル

### 2. React がコンポーネントと Hooks を呼び出す

**重要度**: ⭐⭐⭐

- コンポーネント関数を直接呼び出さない（JSX で使用する）
- Hooks を通常の値として渡さない

**❌ 悪い例**:
```jsx
// コンポーネントを直接呼び出している
return <div>{MyComponent()}</div>;
```

**✅ 良い例**:
```jsx
// JSX として使用
return <div><MyComponent /></div>;
```

### 3. Hooks のルール

**重要度**: ⭐⭐⭐

#### ルール 1: Hooks はトップレベルでのみ呼び出す

**❌ 悪い例**:
```jsx
function Form() {
  if (condition) {
    useState(0);  // 条件付きで呼び出してはいけない
  }

  for (let i = 0; i < 10; i++) {
    useEffect(() => {});  // ループ内で呼び出してはいけない
  }
}
```

**✅ 良い例**:
```jsx
function Form() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    // ...
  });
}
```

#### ルール 2: Hooks は React 関数内でのみ呼び出す

Hooks を呼び出せる場所:
- React コンポーネント内
- カスタム Hook 内

**❌ 悪い例**:
```jsx
function normalFunction() {
  const [count, setCount] = useState(0);  // React 関数外で呼び出してはいけない
}
```

---

## まとめ

### 必ず守るべき原則

1. **コンポーネントを純粋に保つ** - 同じ入力に対して同じ出力を返す
2. **Effect は最後の手段** - 多くの場合、Effect は不要
3. **State 構造を慎重に設計** - 冗長性と重複を避ける
4. **Hooks のルールを守る** - トップレベルでのみ呼び出す
5. **Props と State を直接変更しない** - 常に新しいオブジェクト/配列を作成

### 推奨される最適化手法

1. **useMemo** - 高コストな計算のキャッシュ
2. **memo** - コンポーネントの再レンダリングスキップ
3. **useCallback** - 関数のメモ化
4. **React Compiler** - 自動最適化（React 19+）

### 効果的なコード構造

1. **カスタム Hook** - ロジックの再利用
2. **Reducer + Context** - 複雑な State 管理
3. **コンポーネント分割** - 責任の明確化
4. **フラットな State 構造** - 更新の容易さ

---

## 参考リソース

- [React 公式ドキュメント](https://react.dev)
- [React のルール](https://react.dev/reference/rules)
- [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- [Keeping Components Pure](https://react.dev/learn/keeping-components-pure)
- [Choosing the State Structure](https://react.dev/learn/choosing-the-state-structure)

---

最終確認日: 2026年1月3日
