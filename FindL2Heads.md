
该函数用于reorg时选取合适的L2 Unsafe/Safe/Finalized block。


核心过程如下：

首先通过`currentHeads`从EL获取当前的L2 Unsafe/Safe/Finalized block作为起点。

一般情况下最终返回的L2 Finalized block就是EL返回的原样，除非这两个corner case:
1. 当前处于syncStatusFinishedELButNotFinalized状态：[source](https://github.com/ethereum-optimism/optimism/blob/232c54d44ba206220a9997b076c829d6000c79f5/op-node/rollup/sync/start.go#L125)。
2. EL异常：[source](https://github.com/ethereum-optimism/optimism/blob/232c54d44ba206220a9997b076c829d6000c79f5/op-node/rollup/sync/start.go#L134)。

所以大多数时候，这个函数只会修改EL返回的L2 Unsafe/Safe block。

最终想找到的L2 Safe block是沿着L2 Unsafe block往前回溯时第一个满足这些要求的block：
1. 假设它的L1 origin是block N，它自己是以block N为L1 origin的最后一个L2 block。
2. L1 origin在[N+1, N+2+SyncLookback]范围内的L2 block的L1 origin都是`canonical`的（在最长链上）。
   1. 对应的第一个满足L1 origin是`N+2+SyncLookback`的L2 block叫它`candidate`。
3. 如果回溯到L2 Finalized block还是没找到，就以L2 Finalized block作为L2 Safe block。
4. 整个过程最多回溯`MaxReorgSeqWindows x SyncLookback`个L1 block，如果还没找到，就返回错误。


然后L2 Unsafe block U有2种可能：
1. 跟`candidate`是同一个。或者
2. (candidate, U]范围内的L2 block的L1 origin都是当前L1无法找到的([`ahead`](https://github.com/ethereum-optimism/optimism/blob/3dbcee1c312881166bd544fa857f25d86a20bcd2/op-node/rollup/sync/start.go#L180))。

Rationale：

https://github.com/ethereum-optimism/optimism/pull/11872#issuecomment-2347904524
