
`Position.ToGIndex`的[实现](https://github.com/ethereum-optimism/optimism/blob/a1df3e127b82c4d290fd568bc7e1680145dab55c/op-challenger/game/fault/types/position.go#L146)，用更直观的表示是这个：

```
return uint64(1<<p.depth | p.indexAtDepth)
```

奇妙的是，用这种编号方式后，二叉树中的节点自顶向下刚好形成了从1开始的连续编号：

![gindex](https://specs.optimism.io/static/assets/attack.png)

为何会这样？

下面按数学归纳法证明。

首先若`p.depth=0`，则显然成立。

假设当`p.depth<=k`时，命题成立，当`p.depth=k+1`时：

上面`k+1`层的元素个数一共是`1+2+4+...+2^k=2^(k+1)-1`，按归纳假设，最后一个元素gindex是`2^(k+1)-1`，

跟层k+1的第一个元素的gindex刚好连续，因此仍然成立。

根据数学归纳法，命题始终成立。