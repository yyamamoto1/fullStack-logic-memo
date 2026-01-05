# TODOアプリ ⭐

**難易度**: 初級
**推奨時間**: 8-10時間
**技術スタック**: React, TypeScript, localStorage

---

## 📋 概要

ReactとTypeScriptを使って、基本的なTODOアプリを作成します。状態管理の基礎とローカルストレージを使ったデータ永続化を学びます。

---

## 🎯 学習目標

この演習を完了すると、以下のスキルが身につきます:

- [ ] React Hooks（useState, useEffect）の使い方
- [ ] TypeScriptでのコンポーネント作成
- [ ] 状態管理の基礎
- [ ] localStorageによるデータ永続化
- [ ] フォーム操作とイベントハンドリング
- [ ] 配列操作（map, filter）

---

## 📝 機能要件

### 必須機能（Must Have）

1. **タスクの追加**
   - テキスト入力でタスクを追加
   - 空のタスクは追加できない
   - Enterキーで追加可能

2. **タスクの表示**
   - タスク一覧を表示
   - 完了/未完了の状態を表示
   - タスク数を表示

3. **タスクの完了/未完了**
   - チェックボックスで完了状態を切り替え
   - 完了タスクは取り消し線

4. **タスクの削除**
   - 個別のタスクを削除
   - 完了タスクを一括削除

5. **データ永続化**
   - localStorageに保存
   - ページリロード後もデータが残る

### 追加機能（Should Have）

6. **フィルター機能**
   - 全て/未完了/完了で絞り込み

7. **編集機能**
   - タスク名をダブルクリックで編集

---

## 📐 設計

### データ構造

```typescript
interface Todo {
  id: string;
  text: string;
  completed: boolean;
  createdAt: Date;
}

type FilterType = 'all' | 'active' | 'completed';
```

### コンポーネント構成

```
App
├── Header
│   └── TodoInput
├── TodoList
│   └── TodoItem (複数)
└── Footer
    ├── TodoCount
    └── FilterButtons
```

---

## 💻 実装手順

### Step 1: プロジェクト作成

```bash
npx create-react-app todo-app --template typescript
cd todo-app
npm start
```

### Step 2: コンポーネント作成

1. `TodoInput`: タスク入力フォーム
2. `TodoItem`: 個別のタスク表示
3. `TodoList`: タスク一覧
4. `TodoFilter`: フィルターボタン

### Step 3: 状態管理

```typescript
const [todos, setTodos] = useState<Todo[]>([]);
const [filter, setFilter] = useState<FilterType>('all');
```

### Step 4: localStorage連携

```typescript
useEffect(() => {
  const saved = localStorage.getItem('todos');
  if (saved) {
    setTodos(JSON.parse(saved));
  }
}, []);

useEffect(() => {
  localStorage.setItem('todos', JSON.stringify(todos));
}, [todos]);
```

---

## ✅ 受け入れ基準

- [ ] タスクを追加できる
- [ ] タスク一覧が表示される
- [ ] タスクの完了/未完了を切り替えられる
- [ ] タスクを削除できる
- [ ] ページをリロードしてもデータが残る
- [ ] フィルター機能が動作する
- [ ] TypeScriptの型エラーがない
- [ ] レスポンシブデザイン

---

## 🧪 テスト項目

```typescript
describe('TodoApp', () => {
  it('新しいタスクを追加できる');
  it('空のタスクは追加できない');
  it('タスクを完了にできる');
  it('タスクを削除できる');
  it('完了タスクを一括削除できる');
  it('フィルターが機能する');
  it('localStorageにデータが保存される');
});
```

---

## 📚 参考資料

- [React公式ドキュメント](https://react.dev/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)
- [localStorage API](https://developer.mozilla.org/ja/docs/Web/API/Window/localStorage)

---

**最終更新**: 2025-10-22
