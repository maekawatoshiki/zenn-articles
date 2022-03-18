---
title: "RABasic"
---

このチャプターのタイトルにもなっている [RABasic](https://github.com/llvm/llvm-project/blob/1f001b25f14e0b200ea3ded85e7bd3c046ad0a91/llvm/lib/CodeGen/RegAllocBasic.cpp#L57) は、レジスタアロケータとしての最小限の実装を提供するクラスです。
このクラスは `MachineFunctionPass` を継承しているため、`runOnMachineFunction` メソッドから読み始めると動作を追いやすいです。

# [`RABasic::runOnMachineFunction`](https://github.com/llvm/llvm-project/blob/1f001b25f14e0b200ea3ded85e7bd3c046ad0a91/llvm/lib/CodeGen/RegAllocBasic.cpp#L308)

このメソッド内では色々なことが行われていますが、ここでは `RegAllocBase::allocatePhysRegs` にのみ注目すれば良いです。

## [`RegAllocBase::allocatePhysRegs`](https://github.com/llvm/llvm-project/blob/1df3a913efc497b2ce63d856b25ba8903378d377/llvm/lib/CodeGen/RegAllocBase.cpp#L84)

このメソッドは `RegAllocBase` に実装されています。つまり、他の種類のレジスタアロケータからも呼ばれるということですね。
`allocatePhysRegs` という名前からもわかる通り、物理レジスタを（仮想レジスタに）割り当てるというレジスタアロケータそのものな動作をするメソッドです。

このメソッド内で注目するべきは、`seedLiveRegs`, `selectOrSplit`, ...(TODO:他にもあるよね)... です。

### [`RegAllocBase::seedLiveRegs`](https://github.com/llvm/llvm-project/blob/1df3a913efc497b2ce63d856b25ba8903378d377/llvm/lib/CodeGen/RegAllocBase.cpp#L71)

coming soon...
