# Building a Decentralized Task Management System on ICP

## Table of Contents

1. [Introduction](#introduction)
2. [Overview of the System](#overview-of-the-system)
3. [Prerequisites](#prerequisites)
4. [Defining the Task Model](#defining-the-task-model)
5. [Implementing Task Management Functions](#implementing-task-management-functions)
    - [Create Task](#create-task)
    - [Get All Tasks](#get-all-tasks)
    - [Get Task by ID](#get-task-by-id)
    - [Assign Task](#assign-task)
    - [Complete Task](#complete-task)
    - [Get Tasks by Assignee](#get-tasks-by-assignee)
    - [Get Tasks by Poster](#get-tasks-by-poster)
    - [Get Tasks by Status](#get-tasks-by-status)
    - [Update Task](#update-task)
    - [Delete Task](#delete-task)
6. [Authorization and Security](#authorization-and-security)
7. [Conclusion](#conclusion)

## Introduction

The Internet Computer Protocol (ICP) provides a robust platform for building decentralized applications (dApps). In this article, we will create a decentralized task management system on ICP. This system allows users to create, update, complete, and delete tasks. Additionally, it ensures authorization checks, ensuring only the task owner can modify or delete their tasks.

## Overview of the System

This decentralized task management system offers the following features:

- **Task Creation:** Users can create tasks with a title, description, and reward.
- **Task Retrieval:** Retrieve all tasks, tasks by specific criteria (e.g., assigned user or status), or individual tasks by ID.
- **Task Updates:** Modify task details, mark tasks as completed, or assign them to users.
- **Task Deletion:** Remove tasks securely with proper authorization checks.
- **Authorization:** Only task owners can modify or delete their tasks.

## Prerequisites

Before getting started, ensure you have the following:

- **Node.js**: Installed on your system.
- **Azle Framework**: A JavaScript/TypeScript SDK for developing on ICP.
- **UUID Library**: For generating unique task identifiers.
- **Basic Knowledge**: Familiarity with JavaScript/TypeScript and blockchain concepts.

## Defining the Task Model

The task model defines the structure of tasks managed by the system. It is implemented using Azle's `Record` type:

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

## Implementing Task Management Functions

### Create Task

Users can create tasks with a title, description, and reward. Each task is associated with a unique identifier.

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
    return Ok(`Task "${task.title}" created successfully.`);
});
```


