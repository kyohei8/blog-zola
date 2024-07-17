+++
title="React + TypeScript等ではまったこと"
[taxonomies]
tags=["React", "TypeScript"]
+++

TypeScript で React の開発をしているときにハマった箇所のメモ。

## React のイベントの Props 定義

クリックイベントは`React.MouseEvent`、選択リスト/テキストのインプットは`React.ChangeEvent`が型になる。

```ts
export interface IMyComponentProps {
  prop1: string;
  prop2: (event: React.MouseEvent) => void; // ← 🈁
}

export class MyComponent extends React.Component<IMyComponentProps> {
  render(): JSX.Element {
    const { prop1, prop2 } = this.props;

    return (
      //My code here
      <button onClick={prop2}>{prop1}</button>
    );
  }
}
```

## creatRef の書き方

```ts
class MyComponent extends React.Component<{}, {} > {
  input: React.RefObject<HTMLInputElement>;

  constructor(props: InputTextProps) {
    super(props);
    this.input = React.createRef();
  }
  ...
}

```

## JavaScript の Google Map API の型定義

```bash
$ npm install  @types/googlemaps -D
```

```ts
import {} from 'googlemaps';

const map: google.maps.Map = new google.maps.Map(...);
```

> "message": "ファイル '.../node_modules/@types/googlemaps/index.d.ts' はモジュールではありません。",

というエラーがでるので

`types/index.d.ts` に以下を追加

```ts
declare module 'googlemaps';
```

## Promise を返す関数

おまけですが。

[Returning a promise in an async function in TypeScript - Stack Overflow](https://stackoverflow.com/questions/43881192/returning-a-promise-in-an-async-function-in-typescript)

```ts
const whatever2 = async (): Promise<number> => {
  return new Promise<number>((resolve) => {
    resolve(4);
  });
};
```
