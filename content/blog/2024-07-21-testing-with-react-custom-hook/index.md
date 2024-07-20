+++
title="Reactのカスタムフックテストでハマった話"
[taxonomies]
tags=["React"]
+++

React のカスタムフックのテストでハマったので備忘録として書きます。

## 現象

以下のようなオブジェクトの配列の state とそれを更新する関数を持つカスタムフックを作成しました。

```js
export function useCheckState() {
  const [data, setData] = useState([
    { id: 'foo', isChecked: true },
    { id: 'bar', isChecked: false }
  ]);

  const resetState = () => {
    setData((prev) =>
      prev.map((d) => ({
        ...d,
        isChecked: false
      }))
    );
  };

  return { data, resetState };
}
```

このフックをテストしようとした際、関数 `resetState`を実行したあとで、状態更新が反映されない問題が発生しました。

以下がテストコードです。

```js
import { renderHook, act } from '@testing-library/react-hooks';
import { useCheckState } from './useCheckState';

test('should reset state correctly', () => {
  const {
    result: { current }
  } = renderHook(() => useCheckState());

  act(() => {
    current.resetState();
  });

  expect(current.data).toEqual([
    { id: 'foo', isChecked: false },
    { id: 'bar', isChecked: false }
  ]);
});
```

この expect の部分で ‘foo’の isChecked が true となりテストが失敗する状態となりました。

## 原因

ここで、`current`を直接参照すると最新の状態が取得できないようになってしまいます。

正しくは以下のようになります。

```js
import { renderHook, act } from '@testing-library/react-hooks';
import { useCheckState } from './useCheckState';

test('should reset state correctly', () => {
  const { result } = renderHook(() => useCheckState());

  act(() => {
    result.current.resetState();
  });

  expect(result.current.data).toEqual([
    { id: 'foo', isChecked: false },
    { id: 'bar', isChecked: false }
  ]);
});
```

冷静に考えるとその通りなのだとわかるのですが、分割代入を多用しすぎるとたまにこういう罠にはまってしまうので参照している先を意識するよう心がけたいと思います。
