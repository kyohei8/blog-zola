+++
title="TypeScriptで配列型の要素がオブジェクトの場合にその型を取得する方法"
[taxonomies]
tags=["TypeScript"]
+++

TypeScript でライブラリ内や自動的に生成された型定義で、配列の要素がオブジェクトであるときに、その要素が型として定義されていない場合があります。この型を呼出したり型として再定義する方法です。

```ts
export type Animals = { name: string; size: number }[];

// または exportされていない場合

type Animals = { name: string; size: number };
export type Animals = Animal[];
```

上記のような配列の中身の型を取得したい場合、 `[number]` で型定義を取得できる

```ts
type NewAnimal = Animals[number];
// NewAnimal = {name: string; size: number}
```

`[number]` ではあるため `[0]`や `[1000]` でも一応は問題ない。

ちなみに型のない場合に取得したい場合は以下のようにできる

```ts
const animals = [
  { name: 'dag', size: 10 },
  { name: 'cat', size: 6 }
];

type NewAnimal = (typeof animals)[0];
```
