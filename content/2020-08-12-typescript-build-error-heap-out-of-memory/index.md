+++
title="TypeScriptのビルド中にJavaScript heap out of memoryのエラーが出た"
[taxonomies]
tags=["Node.js", "TypeScript"]
+++

AWS の EC2 のインスタンスが小さいせいか、あるタイミングから tsc コマンドを実行した際に以下のエラーが発生するようになった。

```sh
[stdout]
[stdout]
[stdout]<--- JS stacktrace --->
[stdout]
[stdout]==== JS stack trace =========================================
[stdout]
[stdout] 0: ExitFrame [pc: 0x13725d9]
[stdout]Security context: 0x0b8cca5808a1 <JSObject>
[stdout] 1: getConstraintOfTypeParameter(aka getConstraintOfTypeParameter) [0x750c66c4ea1] [/home/ec2-user/workspace/node_modules/typescript/lib/tsc.js:~35452] [pc=0x27a0e88f956e](this=0x09f8019404a9 <undefined>,0x2f4e5bad5539 <Type map = 0x36323f6119b1>)
[stdout] 2: isUnconstrainedTypeParameter(aka isUnconstrainedTypeParameter) [0x750c66cf3e1] [/home/ec2-user/workspace/no...
[stdout]
[stderr]FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed - JavaScript heap out of memory
[stderr]
[stderr]Writing Node.js report to file: report.20200812.010612.29246.0.001.json
[stderr]Node.js report completed
[stderr] 1: 0x9d8da0 node::Abort() [node]
[stderr] 2: 0x9d9f56 node::OnFatalError(char const*, char const*) [node]
[stderr] 3: 0xb37dbe v8::Utils::ReportOOMFailure(v8::internal::Isolate*, char const*, bool) [node]
[stderr] 4: 0xb38139 v8::internal::V8::FatalProcessOutOfMemory(v8::internal::Isolate*, char const*, bool) [node]
[stderr] 5: 0xce34f5 [node]
[stderr] 6: 0xce3b86 v8::internal::Heap::RecomputeLimits(v8::internal::GarbageCollector) [node]
[stderr] 7: 0xcefa1a v8::internal::Heap::PerformGarbageCollection(v8::internal::GarbageCollector, v8::GCCallbackFlags) [node]
[stderr] 8: 0xcf0925 v8::internal::Heap::CollectGarbage(v8::internal::AllocationSpace, v8::internal::GarbageCollectionReason, v8::GCCallbackFlags) [node]
[stderr] 9: 0xcf3338 v8::internal::Heap::AllocateRawWithRetryOrFail(int, v8::internal::AllocationType, v8::internal::AllocationAlignment) [node]
[stderr]10: 0xcb9c67 v8::internal::Factory::NewFillerObject(int, bool, v8::internal::AllocationType) [node]
[stderr]11: 0xfefb9b v8::internal::Runtime_AllocateInYoungGeneration(int, unsigned long*, v8::internal::Isolate*) [node]
[stderr]12: 0x13725d9 [node]
[stderr]/opt/codedeploy-agent/deployment-root/312c7426-277a-4820-869e-df20d63de27f/d-626LRAIH5/deployment-archive/scripts/post_install.sh: line 12: 29235 Aborted npm run build
```

一応バージョンは以下の通り

```
$ node -v
v12.13.0
$ tsc -v
Version 3.8.3
```

実行コマンドは以下の通りで

```sh
$ npm run tsc
```

package.json にある tsc のコマンドを呼び出すだけである。

```js
// package.json
"scripts": {
    "tsc": "tsc"
    ...
}
```

原因としは、node 実行に割り当てているメモリのサイズが足りないようである。

## 解決策

結論から言うと、実行時に `--max_old_space_size` を付与し割り当てるメモリの量を増やすようにします。

```
$ NODE_OPTIONS=--max_old_space_size=4096 npm run tsc
```

`max_old_space_size` はヒープ上のオブジェクトの合計サイズを制限するものらしく、[デフォルトで割り当てているサイズは 2GB 以下らしく](https://support.circleci.com/hc/en-us/articles/360009208393-How-can-I-increase-the-max-memory-for-Node-)、上記の例では 4GB まで利用できようにしています。
これで実行にメモリの足りないということは起きなくなるという認識です。

テスト環境など、小さなインスタンスで実行する場合は有効な手段かと思います。

### 参考

How can I increase the max memory for Node? – CircleCI Support Center : [https://support.circleci.com/hc/en-us/articles/360009208393-How-can-I-increase-the-max-memory-for-Node-](https://support.circleci.com/hc/en-us/articles/360009208393-How-can-I-increase-the-max-memory-for-Node-)
