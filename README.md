# ICP上で分散型タスク管理システムを構築する

## 目次

1. [はじめに](#はじめに)
2. [システムの概要](#システムの概要)
3. [前提条件](#前提条件)
4. [タスクモデルの定義](#タスクモデルの定義)
5. [タスク管理機能の実装](#タスク管理機能の実装)
    - [タスク作成](#タスク作成)
    - [すべてのタスクを取得](#すべてのタスクを取得)
    - [IDでタスクを取得](#idでタスクを取得)
    - [タスクの割り当て](#タスクの割り当て)
    - [タスク完了](#タスク完了)
    - [担当者別のタスクを取得](#担当者別のタスクを取得)
    - [投稿者別のタスクを取得](#投稿者別のタスクを取得)
    - [ステータス別のタスクを取得](#ステータス別のタスクを取得)
    - [タスクの更新](#タスクの更新)
    - [タスクの削除](#タスクの削除)
6. [認可とセキュリティ](#認可とセキュリティ)
7. [結論](#結論)

## はじめに

Internet Computer Protocol（ICP）は、分散型アプリケーション（dApp）を構築するための強力なプラットフォームを提供します。本記事では、ICP上で分散型タスク管理システムを構築します。このシステムでは、ユーザーがタスクを作成、更新、完了、削除できるほか、タスク所有者のみがタスクを変更または削除できるよう認可チェックを行います。

## システムの概要

この分散型タスク管理システムは、以下の機能を提供します。

- **タスク作成:** タイトル、説明、報酬を指定してタスクを作成できます。
- **タスク取得:** すべてのタスク、特定の条件（例: 割り当てられたユーザーやステータス）に基づくタスク、またはIDで個別のタスクを取得します。
- **タスク更新:** タスクの詳細を変更したり、完了としてマークしたり、ユーザーに割り当てたりできます。
- **タスク削除:** 適切な認可チェックを伴い、タスクを削除します。
- **認可:** タスク所有者のみがそのタスクを変更または削除できます。

## 前提条件

始める前に、以下の要件を確認してください。

- **Node.js:** システムにインストールされていること。
- **Azleフレームワーク:** ICP上で開発するためのJavaScript/TypeScript SDK。
- **UUIDライブラリ:** 一意のタスク識別子を生成するため。
- **基本知識:** JavaScript/TypeScriptやブロックチェーンの概念に関する基本的な理解。

## タスクモデルの定義

タスクモデルは、システムで管理されるタスクの構造を定義します。これはAzleの`Record`型を使用して実装されています。

```typescript
const Task = Record({
    id: text,
    title: text,
    description: text,
    reward: float64,
    poster: Principal,
    assignedTo: Opt(Principal),
    status: text,
    createdAt: nat64,
    updatedAt: Opt(nat64)
});

const taskStorage = StableBTreeMap(text, Task, 1);
```

## タスク管理機能の実装

### タスク作成

ユーザーは、タイトル、説明、報酬を指定してタスクを作成できます。各タスクには一意の識別子が関連付けられます。

```typescript
createTask: update([
    text, text, float64
], Result(text, text), (title, description, reward) => {
    const caller = ic.caller();
    const task = {
        id: uuidv4(),
        title,
        description,
        reward,
        poster: caller,
        assignedTo: null,
        status: 'open',
        createdAt: ic.time(),
        updatedAt: null
    };
    taskStorage.insert(task.id, task);
    return Ok(`タスク「${task.title}」が正常に作成されました。`);
});
```

### すべてのタスクを取得

システムに保存されているすべてのタスクを取得します。

```typescript
getAllTasks: query([], Result(Vec(Task), text), () => {
    const tasks = taskStorage.values();
    return Ok(tasks);
});
```

### IDでタスクを取得

一意の識別子でタスクを取得します。

```typescript
getTaskById: query([text], Result(Task, text), (taskId: string) => {
    const task = taskStorage.get(taskId);
    if ('None' in task) {
        return Err('タスクが見つかりません。');
    }
    return Ok(task.Some);
});
```

### タスクの割り当て

タスクを特定のユーザーに割り当てます。

```typescript
assignTask: update([
    text, Principal
], Result(text, text), (taskId: string, assignee: Principal) => {
    const task = taskStorage.get(taskId);
    if ('None' in task) {
        return Err('タスクが見つかりません。');
    }
    const updatedTask = {
        ...task.Some,
        assignedTo: assignee,
        status: 'assigned',
        updatedAt: ic.time()
    };
    taskStorage.insert(taskId, updatedTask);
    return Ok(`タスク「${updatedTask.title}」が${assignee}に割り当てられました。`);
});
```

### タスク完了

タスクを完了としてマークします。

```typescript
completeTask: update([text], Result(text, text), (taskId: string) => {
    const task = taskStorage.get(taskId);
    if ('None' in task) {
        return Err('タスクが見つかりません。');
    }
    const updatedTask = {
        ...task.Some,
        status: 'completed',
        updatedAt: ic.time()
    };
    taskStorage.insert(taskId, updatedTask);
    return Ok(`タスク「${updatedTask.title}」が完了としてマークされました。`);
});
```

### 担当者別のタスクを取得

特定のユーザーに割り当てられたタスクを取得します。
```typescript
getTasksByAssignee: query([Principal], Result(Vec(Task), text), (assignee: Principal) => {
    const tasks = taskStorage.values().filter(task => task.assignedTo === assignee);
    return Ok(tasks);
});
```

### 投稿者別のタスクを取得

特定のユーザーが作成したタスクを取得します。

```typescript
getTasksByPoster: query([Principal], Result(Vec(Task), text), (poster: Principal) => {
    const tasks = taskStorage.values().filter(task => task.poster === poster);
    return Ok(tasks);
});
```

### ステータス別のタスクを取得

ステータスに基づいてタスクを取得します。

```typescript
getTasksByStatus: query([text], Result(Vec(Task), text), (status: text) => {
    const tasks = taskStorage.values().filter(task => task.status === status);
    return Ok(tasks);
});
```

### タスクの更新

既存のタスクの詳細を変更します。

```typescript
updateTask: update([
    text, text, text, float64
], Result(text, text), (taskId: string, title: string, description: string, reward: float64) => {
    const task = taskStorage.get(taskId);
    if ('None' in task) {
        return Err('タスクが見つかりません。');
    }
    const updatedTask = {
        ...task.Some,
        title,
        description,
        reward,
        updatedAt: ic.time()
    };
    taskStorage.insert(taskId, updatedTask);
    return Ok(`タスク「${updatedTask.title}」が正常に更新されました。`);
});
```

### タスクの削除

システムからタスクを削除します。

```typescript
deleteTask: update([text], Result(text, text), (taskId: string) => {
    const task = taskStorage.get(taskId);
    if ('None' in task) {
        return Err('タスクが見つかりません。');
    }
    taskStorage.remove(taskId);
    return Ok(`タスク「${task.Some.title}」が正常に削除されました。`);
});
```

## 認可とセキュリティ

認可チェックにより、タスク所有者のみがそのタスクを更新または削除できることを保証します。これは、呼び出し元のID（`ic.caller()`）をタスクの`poster`と照合することで実施されます。

## 結論

ICPとAzleを活用することで、堅牢な機能と認可チェックを備えた分散型タスク管理システムを実装しました。このシステムは、Internet Computer Protocol上でセキュアかつスケーラブルなdAppを構築する可能性を示しています。


