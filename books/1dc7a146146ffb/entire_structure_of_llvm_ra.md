---
title: "準備"
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

基本ブロックを一列に並べたとき、ある値が定義されてから最後に使われるまでが生存区間です。
このとき、制御フローによっては生存区間が複数に分割されることがあります。（画像では `%a` が該当）
これら区間の一つひとつは `Segment` と呼ばれています。つまり、生存区間（`LiveRange`）は`Segment`のリストです。

![live range](https://storage.googleapis.com/zenn-user-upload/b6cf57e6049d-20220318.png)

### `SlotIndex` クラス

[`SlotIndexes.h`](https://github.com/llvm/llvm-project/blob/6ffb3ad631c5071ce82c8b6c73dd1c88e0452944/llvm/include/llvm/CodeGen/SlotIndexes.h#L82) に定義されており、`Segment` の開始・終了点を表します。（本当はもうちょっと複雑ですが、今はこのくらいの理解で十分です）

### `LiveInterval` クラス

[`LiveInterval.h`](https://github.com/llvm/llvm-project/blob/112aafcaf425dca901690ca823d25607e5795263/llvm/include/llvm/CodeGen/LiveInterval.h#L680) に定義されており、`LiveRange` に加えてどのレジスタ（or スタックスロット）の生存区間を表しているのか、Spill cost はどれほどなのかといった情報を保持します。
Spill cost については後述することになると思いますが、簡単に言えばそのレジスタをスピルすると、どれほどメモリアクセスが増えることになるのかを表します。
複数の基本ブロックにまたがって複数回使用されているようなレジスタの Spill cost は高くなりがちです。

### `LiveIntervalUnion` クラス

[`LiveIntervalUnion.h`](https://github.com/llvm/llvm-project/blob/af4da4f995f8aaaa30aee6fcd598c3c07ecaff89/llvm/include/llvm/CodeGen/LiveIntervalUnion.h#L42) に定義されており、お互いに干渉しない複数の `LiveInterval` をひとまとめにします。お互いに干渉しない点が重要で、これのおかげで一つの `LiveIntervalUnion` に一つの物理レジスタを割り当てることが可能になります。
（例えば、`LiveRange`の説明で用いた画像を例にすると、`%a` と `%b` は互いに干渉するため、一つの `LiveIntervalUnion` に含めることは出来ません。）

この `LiveIntervalUnion` はレジスタアロケータの中でも重要な要素の一つで、実際にレジスタを割り当てている張本人と言っても過言ではありません。
ここで少し、実際に割り当てを行っているコードを見てみましょう。（誤解のないように言うと、このコード以外にも割り当てに関係する部分はまだ沢山あります。特に `Greedy` アロケータはこのコード以外の比重が大きいです）

以下からコードを引っ張ってきました。コメントを適宜書き加えています。
https://github.com/llvm/llvm-project/blob/af4da4f995f8aaaa30aee6fcd598c3c07ecaff89/llvm/lib/CodeGen/LiveIntervalUnion.cpp#L154-L189

```cpp
  while (LiveUnionI.valid()) {
    .....
    // LiveUnionI は物理レジスタの生存区間(Segment)のイテレータだと思えば良い。
    // LRI は仮想レジスタの生存区間のイテレータだと思えば良い。
    // 物理レジスタの生存区間と仮想レジスタの生存区間が干渉するか調べる。
    while (LRI->start < LiveUnionI.stop() && LRI->end > LiveUnionI.start()) {
      // 干渉するなら、
      const LiveInterval *VReg = LiveUnionI.value();
      if (VReg != RecentReg && !isSeenInterference(VReg)) {
        .....
        // 干渉したことを記録する！
        InterferingVRegs.push_back(VReg);
        .....
      }
      // 物理レジスタの、次の生存区間へ進める。
      if (!(++LiveUnionI).valid()) {
        ......
      }
    }
    ......
    // 仮想レジスタの、次の生存区間へ進める。
    LRI = LR->advanceTo(LRI, LiveUnionI.start());
    ......
  }
......
```
