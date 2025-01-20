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
