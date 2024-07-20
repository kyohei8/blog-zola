+++
title="コンポーネントの型としてReact.FC(React.FunctionComponent)を使わないほうがいい理由"
[taxonomies]
tags=["React", "TypeScript"]
+++

[React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/docs/basic/getting-started/function_components) を見ていたら、React.FC を非推奨してたので、理由を少し調べてみた。

主な理由はこの CRA の pr に書いてあった。

[https://github.com/facebook/create-react-app/pull/8177](https://github.com/facebook/create-react-app/pull/8177)

## マイナス面として

- 暗黙的な `children` の定義してしまう

  - React.FC でコンポーネントを定義すると、暗黙的に children(ReactNode 型)を取るようになり、想定していなくても、すべてのコンポーネントが children を受け入れることになり、以下のようなコードが存在できる。
    ランタイムエラーとはならないが、本来はミスであり、TS で判定できない。

  ```tsx
  const App: React.FC = () => {
    /*... */
  };
  const Example = () => {
    // <App/> はchildrenを利用しないが、渡せてしまう。
    <App>
      <div>Unwanted children</div>
    </App>;
  };
  ```

- generics をサポートしていない

  - （通常であれば）以下のようなコンポーネントを定義することができる

  ```tsx
  type GenericComponentProps<T> = {
    prop: T;
    callback: (t: T) => void;
  };
  const GenericComponent = <T,>(props: GenericComponentProps<T>) => {
    /*...*/
  };
  ```

  しかし、React.FC を使用する場合 generic の `T`を保持、返却する仕組みがないため、これらが不可能です。

  ```tsx
  const GenericComponent: React.FC</* ??? */> = <T,>(props: GenericComponentProps<T>) => {
    /*...*/
  };
  ```

- 「名前空間としてのコンポーネントパターン」をより厄介なものにしてしまう

  - コンポーネントを、関連するコンポーネントの名前空間として使用する(通常は `children`)のは、やや一般的なパターンです。

  ```ts
  <Select>
    <Select.Item />
  </Select>
  ```

  これは React.FC でも可能ですが、以下のように厄介なことになります。

  ```ts
  // 依存するコンポーネントの型を定義しないといけない
  const Select: React.FC<SelectProps> & { Item: React.FC<ItemProps> } = (props) => {
    /* ... */
  };

  Select.Item = (props) => {
    /*...*/
  };
  ```

  React.FC がないと問題なく動作します

  ```tsx
  const Select = (props: SelectProps) => {/* ... */}
  Select.Item = (props: ItemProps) => {/* ...*/ }。
  ```

- defaultProps では正しく動作しない

  💡 [defaultProps は非推奨となる](https://github.com/reactjs/rfcs/pull/107) のでこの件は気にしなくてもよい（デフォルト値の実装はは ES のを利用する。）

  - どちらの場合も ES6 のデフォルトの引数を使用する方がおそらく良いので、これはかなり議論の余地のあるポイントですが...

  ```tsx
  type ComponentProps = { name: string };

  const Component = ({ name }: ComponentProps) => <div>{name.toUpperCase()} /* Safe since name is required */</div>;
  Component.defaultProps = { name: 'John' };

  const Example = () => <Component />; /* Safe to omit since name has a default value */
  ```

  これは正しくコンパイルされる。

  React.FC を利用した場合は、少しだけ間違ったものになる

  - `React.FC<{name: string}>` を利用した場合、name はオプションではなく必須となり
  - `React.FC<{name?: string}>` とした場合は toUpperCase()でエラーになってしまう

  なので「内部的には必須、外部的にはオプション」という動作を再現する方法はないです。

  **※ [defaultProps は非推奨となる](https://github.com/reactjs/rfcs/pull/107)のでそもそも気にしなくてもよい（デフォルト値は ES のを利用する。）**

- コードが長くなる(FunctionalComponent を使っている場合は特に）

  - 大きなポイントではないが、 `React.FC`を利用しているよりは短く書ける

  ```tsx
  const C1: React.FC<CProps> = (props) => {};
  const C2 = (props: CProps) => {};
  ```

## プラス面

- 明示的な戻り値の型を提供する

  - `React.FC` の場合は戻り値を明示的に定義できる点であり、以下のように undefined を返す場合は、コンパイル時にエラーとならない。

    ```tsx
    const Component = () => {
      // コンポーネントはundefinedを返すことはできない（`null`のみ）
      return undefined;
    };
    ```

    ただし、この場合、実際には実行時にエラーとして補足されるので、あまり形式的には変わらないかも？

    ```tsx
    // コンポーネントの戻り値が正しくないのでエラーになる
    const Example = () => <Component />;
    ```

## まとめ

- React.VFC, React.FC を利用しない場合は明示的に `JSX.Element` を戻り値の型としてしたほうがよい（個人的には）

  ```tsx
  const App = ({ message }: AppProps): JSX.Element => <div>{message}</div>;
  ```

- とはいえ現行のコードを"無理に"直す必要はない、と思う。（React.FC であろうが、なかろうがそこまで大きい影響ではないはず）
- ただし、型を定義する場合は、React.FC と VFC はちゃんと使い分けたほうがよい
  - ただし、React v18 から、React.FC から children が外れるようです。それにより React.VFC 自体も不要になるのかも？
  - [https://github.com/DefinitelyTyped/DefinitelyTyped/issues/46691](https://github.com/DefinitelyTyped/DefinitelyTyped/issues/46691)
- リプレイス用の codemod もある
  - [https://github.com/gndelia/codemod-replace-react-fc-typescript](https://github.com/gndelia/codemod-replace-react-fc-typescript)
