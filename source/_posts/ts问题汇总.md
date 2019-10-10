---
title: ts问题汇总
date: 2019-07-11 10:02:58
tags:
  - Typescript
  - 前端
categories: 前端
---

# ts问题汇总

简单记录一下平时遇见的 `ts` 问题。

## 断言

断言是告诉编译器这里的类型，前提是你能正确判断这里的类型。

```typescript
// a.ts
export interface option {
    swaggerPath?: string, // swaggerPath 可以是单个swagger文件，也可以是swagger文件夹，包含多个swagger文件
    schemaPath?: string,
    server?: serverOption
}

export interface serverOption {
    port?: number
}

// b.ts
export const startServer = (option) => {
    const { server } = option
    const { port } = server!
    // .....
}
```

此处的 `b.ts` 需要取到option中的port属性，但是在interface定义的时候是可选的，所以这里直接取值会被编译器提示可能从undefined上取值会报错，如果我们确定这里一定能取到值，可以在属性后面加上 `!` 告诉编译器这里一定不是undefined。

## 全局变量

有时候难免会遇到需要全局变量的时候：

1. 在window上面挂载nim属性：

```typescript
declare global {
    interface Window {
        nim: any
    }
}
export {}
```
2. 全局挂载SDK属性：

```typescript
declare namespace SDK{
    let NIM: {
        getInstance: any
    }
}
```

## 泛型

`T extends U ? X : Y` 可以更精确的控制范型：

```typescript
declare function f<T extends boolean>(x: T): T extends true ? string : number;
```