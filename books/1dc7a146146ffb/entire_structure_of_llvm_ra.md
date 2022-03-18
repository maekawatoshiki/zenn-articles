---
title: "LLVMのレジスタアロケータの構造"
---

# LLVMのレジスタアロケータの構造

ここからは、[`llvm/lib/Codegen`](https://github.com/llvm/llvm-project/tree/main/llvm/lib/CodeGen) 下のソースコードを参照します。特に `RegAlloc*.(cpp|h)` という名前のファイルが重要です。

また、LLVM IR は SSA 形式ですが、レジスタアロケータが動作する段階まで来るともはや SSA ではありません。つまり、値が複数の場所で定義されることがあります。

## 準備

### `RegAllocBase` クラス

[`RegAllocBase.h`](https://github.com/llvm/llvm-project/blob/26c95ae38940b5b6ccfc65188ba9931eb51e468e/llvm/lib/CodeGen/RegAllocBase.h#L61) に定義されており、すべてのレジスタアロケータはこのクラスから派生します。特に、すべてのレジスタアロケータは `selectOrSplit` メソッドをオーバーライドしなければなりません。このメソッドについては後述します。

### `LiveRange` クラス

[`LiveInterval.h`](https://github.com/llvm/llvm-project/blob/112aafcaf425dca901690ca823d25607e5795263/llvm/include/llvm/CodeGen/LiveInterval.h#L157)に定義されており、あるレジスタやスタックスロットの生存区間を表現します。
そもそも生存区間が何なのかは、以下の画像を見ればわかると思います。

<!-- ある値が定義されてから最後に使われるまで -->

基本ブロックを一列に並べたとき、ある値が定義されてから最後に使われるまでが生存区間です。
このとき、制御フローによっては生存区間が複数に分割されることがあります。（画像では `%a` が該当）
これら区間の一つひとつは `Segment` と呼ばれています。つまり、生存区間（`LiveRange`）は`Segment`のリストです。

![live range](/images/liverange.png)


